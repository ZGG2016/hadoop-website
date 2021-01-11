# DistCp

## 1、Overview

> DistCp (distributed copy) is a tool used for large inter/intra-cluster copying. It uses MapReduce to effect its distribution, error handling and recovery, and reporting. It expands a list of files and directories into input to map tasks, each of which will copy a partition of the files specified in the source list.

DistCp(分布式复制)是一种**用于大型集群间/集群内复制**的工具。

它使用 MapReduce 来影响它的分发、错误处理和恢复，以及报告。

**它将文件和目录列表展开作为 map 任务的输入，每个任务将复制在源列表中指定的文件的分区**。

> [The erstwhile implementation of DistCp](http://hadoop.apache.org/docs/r1.2.1/distcp.html) has its share of quirks and drawbacks, both in its usage, as well as its extensibility and performance. The purpose of the DistCp refactor was to fix these shortcomings, enabling it to be used and extended programmatically. New paradigms have been introduced to improve runtime and setup performance, while simultaneously retaining the legacy behaviour as default.

DistCp 以前的实现有它的缺点，无论是在用法上，还是在可扩展性和性能上。DistCp 重构的目的是修复这些缺陷，使其能够以编程的方式使用和扩展。引入了新的范例来改善运行时和设置性能，同时保留默认的遗留行为。

> This document aims to describe the design of the new DistCp, its spanking new features, their optimal use, and any deviance from the legacy implementation.

本文档旨在描述新的 DistCp 的设计，它的新特性，它们的最佳使用，以及与历史实现的任何偏差。

## 2、Usage

### 2.1、Basic Usage

> The most common invocation of DistCp is an inter-cluster copy:

DistCp 的最常见的调用是集群间复制:

	bash$ hadoop distcp hdfs://nn1:8020/foo/bar \
	hdfs://nn2:8020/bar/foo

> This will expand the namespace under /foo/bar on nn1 into a temporary file, partition its contents among a set of map tasks, and start a copy on each NodeManager from nn1 to nn2.

这将把 nn1 上的 `/foo/bar` 下的命名空间扩展成一个临时文件，将其内容划分到一组 map 任务中，并在每个 NodeManager 上从 nn1 到 nn2 开始一个复制。

> One can also specify multiple source directories on the command line:

也**可以在命令行中指定多个源目录**:

	bash$ hadoop distcp hdfs://nn1:8020/foo/a \
	hdfs://nn1:8020/foo/b \
	hdfs://nn2:8020/bar/foo

> Or, equivalently, from a file using the -f option:

或者，**等价使用 `-f` 选项**:

	bash$ hadoop distcp -f hdfs://nn1:8020/srclist \
	hdfs://nn2:8020/bar/foo

> Where srclist contains

srclist 包含:

	hdfs://nn1:8020/foo/a
	hdfs://nn1:8020/foo/b

> When copying from multiple sources, DistCp will abort the copy with an error message if two sources collide, but collisions at the destination are resolved per the [options](https://hadoop.apache.org/docs/r3.2.1/hadoop-distcp/DistCp.html#Command_Line_Options) specified. By default, files already existing at the destination are skipped (i.e. not replaced by the source file). A count of skipped files is reported at the end of each job, but it may be inaccurate if a copier failed for some subset of its files, but succeeded on a later attempt.

当从多个源进行复制时，**如果两个源发生冲突，DistCp 将中止复制**，并发出错误消息，但是在目的地的冲突将根据指定的选项解决。

默认情况下，已经存在于目的地的文件将被跳过(即不被源文件替换)。在每个作业结束时报告跳过文件的计数，但如果复制器对其文件的某些子集复制失败，但在随后的尝试中成功，则可能不准确。

> It is important that each NodeManager can reach and communicate with both the source and destination file systems. For HDFS, both the source and destination must be running the same version of the protocol or use a backwards-compatible protocol; see [Copying Between Versions](#Copying_Between_Versions_of_HDFS).

**每个 NodeManager 能够访问源文件系统和目标文件系统，并与之通信**，这一点非常重要。对于 HDFS，源端和目标端必须运行相同版本的协议，或者使用向后兼容的协议。

> After a copy, it is recommended that one generates and cross-checks a listing of the source and destination to verify that the copy was truly successful. Since DistCp employs both Map/Reduce and the FileSystem API, issues in or between any of the three could adversely and silently affect the copy. Some have had success running with -update enabled to perform a second pass, but users should be acquainted with its semantics before attempting this.

在复制之后，建议生成并交叉检查源和目标的列表，以验证复制是否真正成功。由于 DistCp 同时使用了 Map/Reduce 和 FileSystem API，这三者之间的任何一个问题都可能对复制产生负面的、静默的影响。有些人已经成功运行了启用 `-update` 来执行第二次传递，但是用户在尝试之前应该熟悉它的语义。

> It’s also worth noting that if another client is still writing to a source file, the copy will likely fail. Attempting to overwrite a file being written at the destination should also fail on HDFS. If a source file is (re)moved before it is copied, the copy will fail with a FileNotFoundException.

同样值得注意的是，如果另一个客户端仍在写入源文件，那么**复制可能会失败**。在 HDFS 上试图覆盖正在写入目的地的文件也会失败。如果源文件在复制之前被(重新)移动，则复制将失败，并出现 FileNotFoundException 异常。

> Please refer to the detailed Command Line Reference for information on all the options available in DistCp.

### 2.2、Update and Overwrite

> -update is used to copy files from source that don’t exist at the target or differ from the target version. -overwrite overwrites target-files that exist at the target.

`-update` 用于从在目标不存在或与目标版本不一致的源文件中复制文件。

`-overwrite` 覆盖目标中存在的目标文件。

> The Update and Overwrite options warrant special attention since their handling of source-paths varies from the defaults in a very subtle manner. Consider a copy from /source/first/ and /source/second/ to /target/, where the source paths have the following contents:

`Update` 和 `Overwrite` 选项需要特别注意，因为它们对源路径的处理与默认路径有细微的不同。

考虑一个从 `/source/first/` 和 `/source/second/` 到 `/target/` 的复制，其中源路径包含以下内容:

	hdfs://nn1:8020/source/first/1
	hdfs://nn1:8020/source/first/2
	hdfs://nn1:8020/source/second/10
	hdfs://nn1:8020/source/second/20

> When DistCp is invoked without -update or -overwrite, the DistCp defaults would create directories first/ and second/, under /target. Thus:

当调用 DistCp 而**不使用 `-update` 或 `-overwrite`** 时，DistCp 默认值将在 `/target` 下创建 `first/` 和 `second/` 目录。因此:

	distcp hdfs://nn1:8020/source/first hdfs://nn1:8020/source/second hdfs://nn2:8020/target

> would yield the following contents in /target:

在 `/target` 中，将产生如下内容：

	hdfs://nn2:8020/target/first/1
	hdfs://nn2:8020/target/first/2
	hdfs://nn2:8020/target/second/10
	hdfs://nn2:8020/target/second/20

> When either -update or -overwrite is specified, the contents of the source-directories are copied to target, and not the source directories themselves. Thus:

当指定 `-update` 或 `-overwrite` 时，**源目录的内容**将复制到目标目录，而不是源目录本身。因此:

	distcp -update hdfs://nn1:8020/source/first hdfs://nn1:8020/source/second hdfs://nn2:8020/target

> would yield the following contents in /target:

在 `/target` 中，将产生如下内容：

	hdfs://nn2:8020/target/1
	hdfs://nn2:8020/target/2
	hdfs://nn2:8020/target/10
	hdfs://nn2:8020/target/20

> By extension, if both source folders contained a file with the same name (say, 0), then both sources would map an entry to /target/0 at the destination. Rather than to permit this conflict, DistCp will abort.

通过扩展，**如果两个源文件夹都包含一个具有相同名称(例如，0)的文件**，那么两个源都将一个条目映射到目的地的`/target/0`。与其允许此冲突，DistCp 将中止。

> Now, consider the following copy operation:

现在，考虑下面的复制操作:

	distcp hdfs://nn1:8020/source/first hdfs://nn1:8020/source/second hdfs://nn2:8020/target

With sources/sizes:

	hdfs://nn1:8020/source/first/1 32
	hdfs://nn1:8020/source/first/2 32
	hdfs://nn1:8020/source/second/10 64
	hdfs://nn1:8020/source/second/20 32

And destination/sizes:

	hdfs://nn2:8020/target/1 32
	hdfs://nn2:8020/target/10 32
	hdfs://nn2:8020/target/20 64

Will effect:

	hdfs://nn2:8020/target/1 32
	hdfs://nn2:8020/target/2 32
	hdfs://nn2:8020/target/10 64
	hdfs://nn2:8020/target/20 32

> 1 is skipped because the file-length and contents match. 2 is copied because it doesn’t exist at the target. 10 and 20 are overwritten since the contents don’t match the source.

**1 被跳过，因为文件长度和内容匹配。复制 2 是因为它在目标中不存在。10 和 20 被覆盖，因为其内容与源不匹配**。

> If -update is used, 1 is skipped because the file-length and contents match. 2 is copied because it doesn’t exist at the target. 10 and 20 are overwritten since the contents don’t match the source. However, if -append is additionally used, then only 10 is overwritten (source length less than destination) and 20 is appended with the change in file (if the files match up to the destination’s original length).

如果使用 `-update`，则跳过 1，因为文件长度和内容匹配。复制 2 是因为它在目标中不存在。10 和 20 被覆盖，因为内容与源不匹配。但是，如果附加使用 `-append`，则只覆盖 10(源长度小于目标长度)，并追加 20(如果文件与目标的原始长度匹配)。

> If -overwrite is used, 1 is overwritten as well.

如果使用 `-overwrite`，1 也会被覆盖。

### 2.3、Sync

> -diff option syncs files from a source cluster to a target cluster with a snapshot diff. It copies, renames and removes files in the snapshot diff list.

`-diff` 选项用于将具有快照差的文件从源集群同步到目标集群，对快照差列表中的文件进行复制、重命名和删除。

> -update option must be included when -diff option is in use.

当使用 `-diff` 选项时，必须包含 `-update` 选项。

> Most cloud providers don’t work well with sync at the moment.

目前，大多数云服务提供商都不能很好地实现同步。

用法：

	hadoop distcp -update -diff <from_snapshot> <to_snapshot> <source> <destination>

例如:

	hadoop distcp -update -diff snap1 snap2 /src/ /dst/

> The command above applies changes from snapshot snap1 to snap2 (i.e. snapshot diff from snap1 to snap2) in /src/ to /dst/. Obviously, it requires /src/ to have both snapshots snap1 and snap2. But the destination /dst/ must also have a snapshot with the same name as <from_snapshot>, in this case snap1. The destination /dst/ should not have new file operations (create, rename, delete) since snap1. Note that when this command finishes, a new snapshot snap2 will NOT be created at /dst/.

**上面的命令应用 `/src/` 中的快照 snap1 到 snap2 的更改(即快照snap1和snap2的差异)到 `/dst/`**。

显然，它**要求 `/src/` 同时具有快照 snap1 和 snap2**。

但是**目标 `/dst/` 还必须有一个名称与 `<from_snapshot>` 相同的快照**，在本例中是 snap1。

从 snap1 开始，目标 `/dst/` 不应该有新的文件操作(创建、重命名、删除)。

注意，该**命令执行完毕后，不会在 `/dst/` 创建新的快照 snap2**。

> -update is required to use -diff option.

`-update` 必须使用 `-diff` 选项。

> For instance, in /src/, if 1.txt is added and 2.txt is deleted after the creation of snap1 and before creation of snap2, the command above will copy 1.txt from /src/ to /dst/ and delete 2.txt from /dst/.

例如，在 `/src/` 中，如果在创建 snap1 之后和创建 snap2 之前添加了 `1.txt`，删除了 `2.txt`，上面的命令将把 `1.txt` 从 `/src/` 复制到 `/dst/`，并从 `/dst/` 删除 `2.txt`。

> Sync behavior will be elaborated using experiments below.

同步行为将在下面通过实验进行阐述。

> Experiment 1: Syncing diff of two adjacent snapshots

实验1：同步两个相邻快照的差异

> Some preparations before we start.

开始前做一些准备工作。

	# Create source and destination directories
	hdfs dfs -mkdir /src/ /dst/

	# Allow snapshot on source
	hdfs dfsadmin -allowSnapshot /src/

	# Create a snapshot (empty one)
	hdfs dfs -createSnapshot /src/ snap1

	# Allow snapshot on destination
	hdfs dfsadmin -allowSnapshot /dst/

	# Create a from_snapshot with the same name
	hdfs dfs -createSnapshot /dst/ snap1

	# Put one text file under /src/
	echo "This is the 1st text file." > 1.txt
	hdfs dfs -put 1.txt /src/

	# Create the second snapshot
	hdfs dfs -createSnapshot /src/ snap2

	# Put another text file under /src/
	echo "This is the 2nd text file." > 2.txt
	hdfs dfs -put 2.txt /src/

	# Create the third snapshot
	hdfs dfs -createSnapshot /src/ snap3

> Then we run distcp sync:

	hadoop distcp -update -diff snap1 snap2 /src/ /dst/

> The command above should succeed. 1.txt will be copied from /src/ to /dst/. Again, -update option is required.

上面的命令应该会成功。`1.txt` 将从 `/src/` 复制到 `/dst/`。同样，`-update` 选项是必需的。

> If we run the same command again, we will get DistCp sync failed exception because the destination has added a new file 1.txt since snap1. That being said, if we remove 1.txt manually from /dst/ and run the sync, the command will succeed.

如果我们再次运行相同的命令，我们将得到 DistCp 同步失败的异常，因为目标在 snap1 之后添加了一个新的文件 `1.txt`。也就是说，如果我们手动从 `/dst/` 中删除 `1.txt` 并运行同步，命令将会成功。

> Experiment 2: syncing diff of two non-adjacent snapshots

实验2：同步两个不相邻快照的差异

> First do a clean up from Experiment 1.

首先对实验 1 做一个清理。

	hdfs dfs -rm -skipTrash /dst/1.txt

> Run sync command, note the <to_snapshot> has been changed from snap2 in Experiment 1 to snap3.

运行 sync 命令，注意已经从实验 1 中的 snap2 更改为 snap3。

	hadoop distcp -update -diff snap1 snap3 /src/ /dst/

> Both 1.txt and 2.txt will be copied to /dst/.

`1.txt` 和 `2.txt` 都将被复制到 `/dst/`。

> Experiment 3: syncing file delete operation

实验3：同步文件删除操作

> Continuing from the end of Experiment 2:

从实验 2 结束处继续:

	hdfs dfs -rm -skipTrash /dst/2.txt
	# Create snap2 at destination, it contains 1.txt
	hdfs dfs -createSnapshot /dst/ snap2

	# Delete 1.txt from source
	hdfs dfs -rm -skipTrash /src/1.txt
	# Create snap4 at source, it only contains 2.txt
	hdfs dfs -createSnapshot /src/ snap4

> Run sync command now:

现在执行同步命令:

	hadoop distcp -update -diff snap2 snap4 /src/ /dst/

> 2.txt is copied and 1.txt is deleted under /dst/.

`2.txt` 被复制，`/dst/` 下的 `1.txt` 被删除。

> Note that, though both /src/ and /dst/ have snapshot with the same name snap2, the snapshots don’t need to have the same content. That means, if you have a 1.txt in /dst/’s snap2 but they have different contents, 1.txt will still be removed from /dst/. The sync command doesn’t check the contents of the files that is going to be deleted. It simply follows the snapshot diff list between <from_snapshot> and <to_snapshot>.

注意，**尽管 `/src/` 和 `/dst/` 都具有名称为 snap2 的快照，但快照不需要具有相同的内容**。

这意味着，如果你在 `/dst/` 的 snap2 中有一个 `1.txt`，但它们有不同的内容，`1.txt` 仍然会从 `/dst/`中删除。

sync 命令不会检查要删除的文件的内容。它只是遵循 `<from_snapshot>` 和 `<to_snapshot>` 之间的快照差值列表。

> Also, if we delete 1.txt from /dst/ before creating snap2 on /dst/ in the steps above, so that /dst/’s snap2 doesn’t have 1.txt before running sync command, the command will still succeed. It won’t throw exception while trying to delete 1.txt from /dst/ which doesn’t exist.

同样，如果我们在上面的步骤中，在 `/dst/` 上创建 snap2 之前，从 `/dst/` 删除 `1.txt`，这样 `/dst/` 的 snap2 在运行同步命令之前就没有 `1.txt`，命令仍然会成功。当试图从 `/dst/`中删除不存在的 `1.txt` 时，它不会抛出异常。

### 2.4、raw Namespace Extended Attribute Preservation

> This section only applies to HDFS.

本节仅针对 HDFS。

> If the target and all of the source pathnames are in the /.reserved/raw hierarchy, then ‘raw’ namespace extended attributes will be preserved. ‘raw’ xattrs are used by the system for internal functions such as encryption meta data. They are only visible to users when accessed through the /.reserved/raw hierarchy.

如果目标和所有源路径名都在 `/.reserved/raw` 层次结构中，那么扩展属性的 ‘raw’ 命名空间将被保留。‘raw’ xattrs 被系统用于内部功能，比如加密元数据。它们只有在访问 `/.reserved/raw` 层次结构时才对用户可见。

> raw xattrs are preserved based solely on whether /.reserved/raw prefixes are supplied. The -p (preserve, see below) flag does not impact preservation of raw xattrs.

原始的 xattrs 仅根据是否提供了 `/.reserved/raw` 前缀来决定是否保存。`-p` (preserve，见下文)标志不会影响原始 xattrs 的保存。

> To prevent raw xattrs from being preserved, simply do not use the /.reserved/raw prefix on any of the source and target paths.

为了防止保存原始的 xattrs，只需在任何源和目标路径上不使用 `/.reserved/raw` 的前缀。

> If the /.reserved/rawprefix is specified on only a subset of the source and target paths, an error will be displayed and a non-0 exit code returned.

如果仅在源和目标路径的子集上指定 `/.reserved/raw` 前缀，将显示错误，并返回非0的退出代码。

## 3、Command Line Options

见原文：[https://hadoop.apache.org/docs/r3.2.1/hadoop-distcp/DistCp.html](https://hadoop.apache.org/docs/r3.2.1/hadoop-distcp/DistCp.html)

## 4、Architecture of DistCp

> The components of the new DistCp may be classified into the following categories:

新 DistCp 的组成部分可分为以下几类:

- DistCp Driver
- Copy-listing generator
- Input-formats and Map-Reduce components

### 4.1、DistCp Driver

> The DistCp Driver components are responsible for:

DistCp 驱动组件负责:

> Parsing the arguments passed to the DistCp command on the command-line, via:

- **在命令行中，解析传递给 DistCp 命令的参数**，方法如下:

	- OptionsParser, and
	- DistCpOptionsSwitch

> Assembling the command arguments into an appropriate DistCpOptions object, and initializing DistCp. These arguments include:

- **将命令参数集成到适当的 DistCpOptions 对象中，并初始化 DistCp**。这些参数包括:

	- Source-paths
    - Target location
	- Copy options (e.g. whether to update-copy, overwrite, which file-attributes to preserve, etc.)例如是否更新-复制，覆盖，保留哪个文件属性，等等

> Orchestrating the copy operation by:

- 通过以下方式**编排复制操作**:

	- 调用 copy-listing-generator 来创建要复制的文件列表。
	- 设置并启动Hadoop Map-Reduce作业来执行复制。
	- 根据这些选项，要么立即返回一个句柄给 Hadoop MR 作业，要么等待完成。

> Invoking the copy-listing-generator to create the list of files to be copied.
> Setting up and launching the Hadoop Map-Reduce Job to carry out the copy.
> Based on the options, either returning a handle to the Hadoop MR Job immediately, or waiting till completion.

- The parser-elements are exercised only from the command-line (or if DistCp::run() is invoked). The DistCp class may also be used programmatically, by constructing the DistCpOptions object, and initializing a DistCp object appropriately.

解析器元素仅从命令行执行(或者调用DistCp::run())。

**通过构造 DistCpOptions 对象，并适当初始化一个 DistCp 对象，DistCp 类也可以以编程方式使用**。

### 4.2、Copy-listing Generator

> The copy-listing-generator classes are responsible for creating the list of files/directories to be copied from source. They examine the contents of the source-paths (files/directories, including wild-cards), and record all paths that need copy into a SequenceFile, for consumption by the DistCp Hadoop Job. The main classes in this module include:

copy-listing-generator 类负责**创建要从源中复制的文件/目录列表**。它们检查源路径的内容(文件/目录，包括通配符)，并记录所有需要复制到 SequenceFile 中的路径，以供 DistCp Hadoop 作业使用。

这个模块中的主要类包括:

> CopyListing: The interface that should be implemented by any copy-listing-generator implementation. Also provides the factory method by which the concrete CopyListing implementation is chosen.

1.CopyListing:任何 copy-listing-generator 都应该实现的接口。还提供了用于选择具体 CopyListing 实现的工厂方法。

> SimpleCopyListing: An implementation of CopyListing that accepts multiple source paths (files/directories), and recursively lists all the individual files and directories under each, for copy.

2.SimpleCopyListing: CopyListing 的一个实现，它接受多个源路径(文件/目录)，递归地列出每个路径下的所有单独文件和目录，以便复制。

> GlobbedCopyListing: Another implementation of CopyListing that expands wild-cards in the source paths.

3.GlobbedCopyListing:在源路径中扩展了通配符的 CopyListing 的另一个实现。

> FileBasedCopyListing: An implementation of CopyListing that reads the source-path list from a specified file.

4.FileBasedCopyListing: CopyListing 的实现，它从指定的文件读取源路径列表。

> Based on whether a source-file-list is specified in the DistCpOptions, the source-listing is generated in one of the following ways:

基于是否在 DistCpOptions 中指定了源文件列表，源文件列表会以以下方式之一生成:

> If there’s no source-file-list, the GlobbedCopyListing is used. All wild-cards are expanded, and all the expansions are forwarded to the SimpleCopyListing, which in turn constructs the listing (via recursive descent of each path).

1.如果没有源文件列表，则使用 GlobbedCopyListing。

所有的通配符都被扩展，所有的扩展都被转发给 SimpleCopyListing，SimpleCopyListing 反过来构造列表(通过每个路径的递归下降)。

> If a source-file-list is specified, the FileBasedCopyListing is used. Source-paths are read from the specified file, and then forwarded to the GlobbedCopyListing. The listing is then constructed as described above.

2.如果指定了源文件列表，则使用 FileBasedCopyListing。源路径从指定的文件中读取，然后转发到 GlobbedCopyListing。然后按照上面的描述构造列表。

> One may customize the method by which the copy-listing is constructed by providing a custom implementation of the CopyListing interface. The behaviour of DistCp differs here from the legacy DistCp, in how paths are considered for copy.

可以通过提供 CopyListing 接口的自定义实现来定制用于构造复制列表的方法。这里的 DistCp 的行为与传统的 DistCp 不同之处在于如何考虑复制路径。

> The legacy implementation only lists those paths that must definitely be copied on to target. E.g. if a file already exists at the target (and -overwrite isn’t specified), the file isn’t even considered in the MapReduce Copy Job. Determining this during setup (i.e. before the MapReduce Job) involves file-size and checksum-comparisons that are potentially time-consuming.

**传统实现只列出那些必须被复制到目标上的路径**。

例如，如果一个文件已经存在于目标(并且没有指定-overwrite)，这个文件甚至不会用在 MapReduce 复制作业中。在设置过程中(即在MapReduce作业之前)确定这个涉及到文件大小和校验和比较，这可能很耗时。

> The new DistCp postpones such checks until the MapReduce Job, thus reducing setup time. Performance is enhanced further since these checks are parallelized across multiple maps.

**新的 DistCp 将这些检查延迟到 MapReduce 作业，从而减少了设置时间。由于这些检查在多个 maps 上并行执行，性能进一步得到增强**。

### 4.3、InputFormats and MapReduce Components

> The InputFormats and MapReduce components are responsible for the actual copy of files and directories from the source to the destination path. The listing-file created during copy-listing generation is consumed at this point, when the copy is carried out. The classes of interest here include:

inputformat 和 MapReduce 组件**负责将文件和目录从源路径实际复制到目标路径**。在复制执行时，在复制列表生成期间创建的列表文件将在此时使用。

这里的类包括:

> UniformSizeInputFormat: This implementation of org.apache.hadoop.mapreduce.InputFormat provides equivalence with Legacy DistCp in balancing load across maps. The aim of the UniformSizeInputFormat is to make each map copy roughly the same number of bytes. Apropos, the listing file is split into groups of paths, such that the sum of file-sizes in each InputSplit is nearly equal to every other map. The splitting isn’t always perfect, but its trivial implementation keeps the setup-time low.

- UniformSizeInputFormat：`org.apache.hadoop.mapreduce.InputFormat` 的实现也提供了同传统 DistCp 一样的 maps 间的平衡负载。UniformSizeInputFormat 的目的是**让每个 map 复制大致相同的字节数**。顺便说一下，列表文件被分割成路径组，这样每个 InputSplit 中的文件大小之和几乎等于每个其他 map。这种切片并不总是完美的，但它的简单实现可以降低设置时间。

> DynamicInputFormat and DynamicRecordReader: The DynamicInputFormat implements org.apache.hadoop.mapreduce.InputFormat, and is new to DistCp. The listing-file is split into several “chunk-files”, the exact number of chunk-files being a multiple of the number of maps requested for in the Hadoop Job. Each map task is “assigned” one of the chunk-files (by renaming the chunk to the task’s id), before the Job is launched. Paths are read from each chunk using the DynamicRecordReader, and processed in the CopyMapper. After all the paths in a chunk are processed, the current chunk is deleted and a new chunk is acquired. The process continues until no more chunks are available. This “dynamic” approach allows faster map-tasks to consume more paths than slower ones, thus speeding up the DistCp job overall.

- DynamicInputFormat 和 DynamicRecordReader：DynamicInputFormat 实现了 `org.apache.hadoop.mapreduce.InputFormat` ，对 DistCp 是新特性。

**列表文件被划分成几个“块文件”**，块文件的确切数量是Hadoop作业中请求的 maps 数量的倍数。

**在作业启动之前，为每个 map 任务“分配”一个块文件**(通过将块重命名为任务的id)。

**使用 DynamicRecordReader 从每个块中读取路径，并在 CopyMapper 中处理**。一个块中的所有路径被处理完后，删除当前块，并获取一个新的块。该过程将继续，直到没有更多的块可用为止。这种“动态”方法允许更快的 map-task 比更慢的 map-task 消耗更多的路径，从而在总体上加速 DistCp 作业。

> CopyMapper: This class implements the physical file-copy. The input-paths are checked against the input-options (specified in the Job’s Configuration), to determine whether a file needs copy. A file will be copied only if at least one of the following is true:

- CopyMapper：这个类**实现了物理文件复制**。根据输入选项(在作业的配置中指定)检查输入路径，以确定文件是否需要复制。只有当以下至少一个为真时，文件才会被复制:

	- 在目标中不存在同名的文件。
	- 在目标中存在同名文件，但文件大小不同。
	- 在目标中存在一个同名的文件，但是有不同的校验和，并且没有 `-skipcrccheck`。
	- 在目标中存在同名文件，但指定了`-overwrite`。
	- 在目标中存在同名文件，但块大小不同(需要保留块大小)。

> A file with the same name doesn’t exist at target.
> A file with the same name exists at target, but has a different file size.
> A file with the same name exists at target, but has a different checksum, and -skipcrccheck isn’t mentioned.
> A file with the same name exists at target, but -overwrite is specified.
> A file with the same name exists at target, but differs in block-size (and block-size needs to be preserved.

> CopyCommitter: This class is responsible for the commit-phase of the DistCp job, including:

- CopyCommitter：这个类**负责 DistCp 任务的提交阶段**，包括:

	- 保存目录权限(如果在选项中指定)
	- 清理临时文件、工作目录等。

> Preservation of directory-permissions (if specified in the options)
> Clean-up of temporary-files, work-directories, etc.

## 5、Appendix

### 5.1、Map sizing

> By default, DistCp makes an attempt to size each map comparably so that each copies roughly the same number of bytes. Note that files are the finest level of granularity, so increasing the number of simultaneous copiers (i.e. maps) may not always increase the number of simultaneous copies nor the overall throughput.

**默认情况下，DistCp 会尝试比较每个 map 的大小，以便每个 map 复制大致相同的字节数**。

请注意，文件是最细的粒度级别，因此增加同时复制器的数量(比如maps)不一定会增加同时复制的数量，也不一定会增加总体吞吐量。

> The new DistCp also provides a strategy to “dynamically” size maps, allowing faster data-nodes to copy more bytes than slower nodes. Using -strategy dynamic (explained in the Architecture), rather than to assign a fixed set of source-files to each map-task, files are instead split into several sets. The number of sets exceeds the number of maps, usually by a factor of 2-3. Each map picks up and copies all files listed in a chunk. When a chunk is exhausted, a new chunk is acquired and processed, until no more chunks remain.

**新的 DistCp 还提供了一种“动态”大小映射的策略，允许更快的数据节点比更慢的节点复制更多字节**。

**使用 `-strategy dynamic`**(在体系结构中有解释)，而不是为每个 map-task 分配一组固定的源文件，而是将文件分成几个集合。集合的数量超过 maps 的数量，通常是 2-3 倍。每个 map 拾取并复制块中列出的所有文件。

当一个块耗尽时，将获取并处理一个新的块，直到没有更多的块留下来。

> By not assigning a source-path to a fixed map, faster map-tasks (i.e. data-nodes) are able to consume more chunks, and thus copy more data, than slower nodes. While this distribution isn’t uniform, it is fair with regard to each mapper’s capacity.

通过不给一个固定的 map 分配一个源路径，更快的 map-tasks(即数据节点)能够比较慢的节点消耗更多的块，从而复制更多的数据。

虽然这种分布不是均匀的，但就每个 mapper 的容量而言，它是公平的。

> The dynamic-strategy is implemented by the DynamicInputFormat. It provides superior performance under most conditions.

**动态策略由 DynamicInputFormat 实现**。它在大多数条件下提供优越的性能。

> Tuning the number of maps to the size of the source and destination clusters, the size of the copy, and the available bandwidth is recommended for long-running and regularly run jobs.

对于长时间运行和定期运行的作业，建议根据源和目标集群的大小、副本的大小和可用带宽，调整 maps 的数量。

### 5.2、Copying Between Versions of HDFS

> For copying between two different major versions of Hadoop (e.g. between 1.X and 2.X), one will usually use WebHdfsFileSystem. Unlike the previous HftpFileSystem, as webhdfs is available for both read and write operations, DistCp can be run on both source and destination cluster. Remote cluster is specified as webhdfs://<namenode_hostname>:<http_port>. When copying between same major versions of Hadoop cluster (e.g. between 2.X and 2.X), use hdfs protocol for better performance.

**对于在两个不同的 Hadoop 主要版本之间复制(例如，在1.X和2.X)，通常会使用 WebHdfsFileSystem**。

与以前的 HftpFileSystem 不同，由于 webhdfs 既可用于读也可用于写操作，因此 DistCp 可以在源集群和目标集群上运行。

远端集群指定为 `webhdfs://<namenode_hostname>:<http_port>`。

**当在 Hadoop 集群的相同主要版本之间复制时(例如在2.X和2.X)，使用 hdfs 协议以获得更好的性能**。

### 5.3、Secure Copy over the wire with distcp

> Use the “swebhdfs://” scheme when webhdfs is secured with SSL. For more information see [SSL Configurations for SWebHDFS](https://hadoop.apache.org/docs/r3.2.1/hadoop-project-dist/hadoop-hdfs/WebHDFS.html#SSL_Configurations_for_SWebHDFS).

**当使用 SSL 保护 webhdfs 时，使用 “swebhdfs://” 方案**。

### 5.4、MapReduce and other side-effects

> As has been mentioned in the preceding, should a map fail to copy one of its inputs, there will be several side-effects.

正如前面提到的，如果一个 map 不能复制其中一个输入，将会有几个副作用。

- 除非指定`-overwrite`，否则前一个 map 在重新执行时成功复制的文件将被标记为“跳过”。

- 如果 map 失败了 `mapreduce.map.maxattempts`次数时，剩余的 map 任务将被杀死(除非设置了`-i`)。

- 如果 `mapreduce.map.speculative` 被设置为 set final 和 true，则复制的结果未定义。

> Unless -overwrite is specified, files successfully copied by a previous map on a re-execution will be marked as “skipped”.
> If a map fails mapreduce.map.maxattempts times, the remaining map tasks will be killed (unless -i is set).
> If mapreduce.map.speculative is set set final and true, the result of the copy is undefined.

### 5.5、DistCp and Object Stores

> DistCp works with Object Stores such as Amazon S3, Azure WASB and OpenStack Swift.

**DistCp 可用于对象存储**，如 Amazon S3、Azure WASB 和 OpenStack Swift。

Prequisites

- 包含对象存储实现的 JAR 及其所有依赖项都在类路径上。

- 除非 JAR 自动注册其绑定的文件系统客户端，否则可能需要修改配置以声明实现文件系统模式的类。所有 ASF 自己的对象存储客户端都是自注册的。

- 相关的对象存储访问凭据必须在集群配置中可用，或者在所有集群主机中可用。

> The JAR containing the object store implementation is on the classpath, along with all of its dependencies.
> Unless the JAR automatically registers its bundled filesystem clients, the configuration may need to be modified to state the class which implements the filesystem schema. All of the ASF’s own object store clients are self-registering.
> The relevant object store access credentials must be available in the cluster configuration, or be otherwise available in all cluster hosts.

> DistCp can be used to upload data

可以**使用 DistCp 上传数据**

	hadoop distcp -direct hdfs://nn1:8020/datasets/set1 s3a://bucket/datasets/set1

> To download data

**下载数据**

	hadoop distcp s3a://bucket/generated/results hdfs://nn1:8020/results

> To copy data between object stores

**在对象存储之间复制数据**

	hadoop distcp s3a://bucket/generated/results wasb://updates@example.blob.core.windows.net

> And do copy data within an object store

并**在对象存储中复制数据**

	hadoop distcp wasb://updates@example.blob.core.windows.net/current \
	wasb://updates@example.blob.core.windows.net/old

> And to use -update to only copy changed files.

**使用 `-update` 只复制修改过的文件**。

	hadoop distcp -update -numListstatusThreads 20  \
	  swift://history.cluster1/2016 \
	  hdfs://nn1:8020/history/2016

> Because object stores are slow to list files, consider setting the -numListstatusThreads option when performing a -update operation on a large directory tree (the limit is 40 threads).

因为对象存储在列出文件时比较慢，所以**在对一个大型目录树执行 `-update`操作时，考虑设置 `-numListstatusThreads` 选项**(限制是40个线程)。

> When DistCp -update is used with object stores, generally only the modification time and length of the individual files are compared, not any checksums. The fact that most object stores do have valid timestamps for directories is irrelevant; only the file timestamps are compared. However, it is important to have the clock of the client computers close to that of the infrastructure, so that timestamps are consistent between the client/HDFS cluster and that of the object store. Otherwise, changed files may be missed/copied too often.

**当 `DistCp -update` 用于对象存储时，通常只比较单个文件的修改时间和长度，而不比较任何校验和**。

事实上，大多数对象存储都有有效的目录时间戳，这是无关紧要的；只比较文件时间戳。但是，让客户端计算机的时钟接近设备的时钟是很重要的，这样客户端/HDFS集群和对象存储的时间戳是一致的。否则，更改的文件可能会被太频繁地错过/复制。

注意：

> The -atomic option causes a rename of the temporary data, so significantly increases the time to commit work at the end of the operation. Furthermore, as Object Stores other than (optionally) wasb:// do not offer atomic renames of directories the -atomic operation doesn’t actually deliver what is promised. Avoid.

- `-atomic` 选项导致对临时数据的重命名，因此在操作结束时，提交工作的时间显著增加。此外，由于对象存储不是(可选的)`wasb://` 不提供目录的原子重命名，`-atomic` 操作实际上并没有交付所承诺的内容。避免的。

> The -append option is not supported.

- 不支持 `-append` 选项。

> The -diff and rdiff options are not supported

- 不支持 `-diff`和 `rdiff` 选项

> CRC checking will not be performed, irrespective of the value of the -skipCrc flag.

- 无论 `-skipCrc` 标志的值是多少，都不会执行 CRC 检查。

> All -p options, including those to preserve permissions, user and group information, attributes checksums and replication are generally ignored. The wasb:// connector will preserve the information, but not enforce the permissions.

- 所有的 `-p` 选项，包括那些保存权限、用户和组信息、属性校验和和复制的选项通常会被忽略。`wasb://`连接器将保留信息，但不强制执行权限。

> Some object store connectors offer an option for in-memory buffering of output —for example the S3A connector. Using such option while copying large files may trigger some form of out of memory event, be it a heap overflow or a YARN container termination. This is particularly common if the network bandwidth between the cluster and the object store is limited (such as when working with remote object stores). It is best to disable/avoid such options and rely on disk buffering.

- 一些对象存储连接器提供了在内存中缓冲输出的选项：例如S3A连接器。在复制大文件时使用此选项可能会触发某种形式的内存溢出事件，可能是堆溢出或YARN容器终止。如果集群和对象存储之间的网络带宽有限(例如使用远程对象存储时)，这种情况尤其常见。最好禁用/避免这些选项，并依赖于磁盘缓冲。

> Copy operations within a single object store still take place in the Hadoop cluster —even when the object store implements a more efficient COPY operation internally

- 单个对象存储中的复制操作仍然发生在 Hadoop 集群中：即使对象存储在内部实现了更有效的复制操作

> That is, an operation such as

也就是说，一个操作，例如

	hadoop distcp s3a://bucket/datasets/set1 s3a://bucket/datasets/set2

> Copies each byte down to the Hadoop worker nodes and back to the bucket. As well as being slow, it means that charges may be incurred.

将每个字节复制到 Hadoop worker 节点，并返回到存储桶。除了速度慢之外，它还意味着可能会产生费用。

> The -direct option can be used to write to object store target paths directly, avoiding the potentially very expensive temporary file rename operations that would otherwise occur.

- 可以使用 `-direct` 选项直接写入对象存储目标路径，避免可能非常昂贵的临时文件重命名操作，否则可能会发生。

## 6、Frequently Asked Questions

> Why does -update not create the parent source-directory under a pre-existing target directory? The behaviour of -update and -overwrite is described in detail in the Usage section of this document. In short, if either option is used with a pre-existing destination directory, the contents of each source directory is copied over, rather than the source-directory itself. This behaviour is consistent with the legacy DistCp implementation as well.

1.为什么 `-update` 不在预先存在的目标目录下创建父源目录?

`-update` 和 `-overwrite` 的行为在本文档的用法部分有详细描述。简而言之，如果对已存在的目标目录使用任何一种选项，则复制每个源目录的内容，而不是复制源目录本身。这种行为也与传统的 DistCp 实现一致。

> How does the new DistCp differ in semantics from the Legacy DistCp?

2.新的 DistCp 与旧的 DistCp 在语义上有什么不同?

- 当使用传统的 DistCp 复制时，在复制过程中被跳过的文件也会有它们的文件属性(权限，所有者/组信息等)不变。即使文件复制被跳过，这些文件也会被更新。

- 在传统的 DistCp 中，源路径输入中的空根目录没有在目标上创建。现在就创建好了。

> Files that are skipped during copy used to also have their file-attributes (permissions, owner/group info, etc.) unchanged, when copied with Legacy DistCp. These are now updated, even if the file-copy is skipped.

> Empty root directories among the source-path inputs were not created at the target, in Legacy DistCp. These are now created.

> Why does the new DistCp use more maps than legacy DistCp? Legacy DistCp works by figuring out what files need to be actually copied to target before the copy-job is launched, and then launching as many maps as required for copy. So if a majority of the files need to be skipped (because they already exist, for example), fewer maps will be needed. As a consequence, the time spent in setup (i.e. before the M/R job) is higher. The new DistCp calculates only the contents of the source-paths. It doesn’t try to filter out what files can be skipped. That decision is put off till the M/R job runs. This is much faster (vis-a-vis execution-time), but the number of maps launched will be as specified in the -m option, or 20 (default) if unspecified.

3.为什么新的 DistCp 比旧的 DistCp 使用更多的 maps?

传统 DistCp 的工作方式是在启动复制作业之前确定哪些文件需要实际复制到目标，然后启动复制所需的尽可能多的 maps。因此，如果需要跳过大部分文件(例如，因为它们已经存在)，则需要的 maps 将更少。因此，花在设置(即在M/R任务之前)上的时间会更高。新的 DistCp 只计算源路径的内容。它不会试图过滤掉哪些文件可以跳过。这个决定被推迟到 M/R 作业运行时。这要快得多(相对于执行时间)，但是启动的 maps 数量将在 `-m` 选项中指定，如果未指定，则为 20(默认值)。

> Why does DistCp not run faster when more maps are specified? At present, the smallest unit of work for DistCp is a file. i.e., a file is processed by only one map. Increasing the number of maps to a value exceeding the number of files would yield no performance benefit. The number of maps launched would equal the number of files.

4.当指定更多的 maps 时，为什么 DistCp 不能运行得更快?

目前，DistCp 的最小工作单元是文件。也就是说，一个文件只被一个 map 处理。将 map 的数量增加到超过文件数量的值不会带来性能上的好处。map 的数量将等于文件的数量。

> Why does DistCp run out of memory? If the number of individual files/directories being copied from the source path(s) is extremely large (e.g. 1,000,000 paths), DistCp might run out of memory while determining the list of paths for copy. This is not unique to the new DistCp implementation. To get around this, consider changing the -Xmx JVM heap-size parameters, as follows:

5.为什么 DistCp 会耗尽内存?

如果从源路径复制的单个文件/目录数量非常大(例如1,000,000个路径)，DistCp 在确定复制的路径列表时可能会耗尽内存。这并不是新的 DistCp 实现所独有的。为了解决这个问题，考虑更改 `-Xmx JVM` 堆大小参数，如下所示:

	 bash$ export HADOOP_CLIENT_OPTS="-Xms64m -Xmx1024m"
	 bash$ hadoop distcp /source /target