# User Guide

[TOC]

## 1、Purpose

> This document is a starting point for users working with Hadoop Distributed File System (HDFS) either as a part of a Hadoop cluster or as a stand-alone general purpose distributed file system. While HDFS is designed to “just work” in many environments, a working knowledge of HDFS helps greatly with configuration improvements and diagnostics on a specific cluster.

对于使用 Hadoop 分布式文件系统(HDFS)作为 Hadoop 集群的一部分或作为独立的通用分布式文件系统的用户来说，本文档是一个起点。

虽然 HDFS 被设计为在许多环境中“只能工作”，但是对于 HDFS 的工作知识对于特定集群的配置的改进和诊断有很大帮助。

## 2、Overview

> HDFS is the primary distributed storage used by Hadoop applications. A HDFS cluster primarily consists of a NameNode that manages the file system metadata and DataNodes that store the actual data. The HDFS Architecture Guide describes HDFS in detail. This user guide primarily deals with the interaction of users and administrators with HDFS clusters. The HDFS architecture diagram depicts basic interactions among NameNode, the DataNodes, and the clients. Clients contact NameNode for file metadata or file modifications and perform actual file I/O directly with the DataNodes.

HDFS 是 Hadoop 应用程序使用的主要的分布式存储。

一个 HDFS 集群主要由一个 NameNode 和 DataNodes 组成，NameNode 负责管理文件系统元数据，DataNodes 负责存储实际数据。

HDFS Architecture Guide 对 HDFS 有详细的描述。本用户指南主要介绍用户和管理员与 HDFS 集群的交互。HDFS 架构图描述了 NameNode、DataNodes 和客户端之间的基本交互。客户端和 NameNode 通信获取文件元数据或修改文件，直接和 DataNodes 通信执行实际的文件I/O。

> The following are some of the salient features that could be of interest to many users.

以下是许多用户可能感兴趣的一些显著特性。

> Hadoop, including HDFS, is well suited for distributed storage and distributed processing using commodity hardware. It is fault tolerant, scalable, and extremely simple to expand. MapReduce, well known for its simplicity and applicability for large set of distributed applications, is an integral part of Hadoop.

- Hadoop（包括HDFS）非常适合使用普通硬件进行分布式存储和分布式处理。

	它具有容错性、可扩展性，并且非常容易扩展。MapReduce 以其简单性和对大型分布式应用的适用性而闻名，它是 Hadoop 不可分割的一部分。

> HDFS is highly configurable with a default configuration well suited for many installations. Most of the time, configuration needs to be tuned only for very large clusters.

- HDFS 是高度可配置的，默认配置非常适合许多安装。大多数情况下，只需要针对非常大的集群调整配置。

> Hadoop is written in Java and is supported on all major platforms.

- Hadoop 是用 Java 编写的，所有主要平台都支持 Hadoop。

> Hadoop supports shell-like commands to interact with HDFS directly.

- Hadoop 支持类似 shell 的命令直接与 HDFS 交互。

> The NameNode and Datanodes have built in web servers that makes it easy to check current status of the cluster.

- NameNode 和 Datanodes 内置在 web 服务中，可以方便地检查集群的当前状态。

> New features and improvements are regularly implemented in HDFS. The following is a subset of useful features in HDFS:

- 新的特性和改进定期在 HDFS 中实现。以下是 HDFS 中一些有用特性的子集:

> File permissions and authentication.

- a. 文件权限和身份验证。

> Rack awareness: to take a node’s physical location into account while scheduling tasks and allocating storage.

- b. 机架感知：在调度任务和分配存储时考虑节点的物理位置。

> Safemode: an administrative mode for maintenance.

- c. Safemode：为了维护的管理模式。

> fsck: a utility to diagnose health of the file system, to find missing files or blocks.

- d. fsck：诊断文件系统健康状况的实用程序，以查找丢失的文件或块。

> fetchdt: a utility to fetch DelegationToken and store it in a file on the local system.

- e. fetchdt：一个获取 DelegationToken 并将其存储在本地系统的文件中的工具。

> Balancer: tool to balance the cluster when the data is unevenly distributed among DataNodes.

- f. Balancer：数据在 DataNodes 间分布不均匀时实现集群均衡的工具。

> Upgrade and rollback: after a software upgrade, it is possible to rollback to HDFS’ state before the upgrade in case of unexpected problems.

- g. Upgrade 和 rollback：软件升级后，如果出现意外问题，可以在升级前回退到 HDFS 的状态。

> Secondary NameNode: performs periodic checkpoints of the namespace and helps keep the size of file containing log of HDFS modifications within certain limits at the NameNode.

- h. Secondary NameNode：执行命名空间的周期性检查点，在 NameNode 上，帮助控制包含 HDFS 修改日志的文件大小在一定的范围内。

> Checkpoint node: performs periodic checkpoints of the namespace and helps minimize the size of the log stored at the NameNode containing changes to the HDFS. Replaces the role previously filled by the Secondary NameNode, though is not yet battle hardened. The NameNode allows multiple Checkpoint nodes simultaneously, as long as there are no Backup nodes registered with the system.

- i. Checkpoint node：执行命名空间的周期性检查点，帮助最小化存储在 NameNode 上的日志大小，其中日志包含了对 HDFS 的更改。取代了以前由 Secondary NameNode 所扮演的角色，尽管还没有经过战斗的磨练。NameNode 允许同时有多个 Checkpoint nodes，只要没有向系统注册 Backup nodes。

> Backup node: An extension to the Checkpoint node. In addition to checkpointing it also receives a stream of edits from the NameNode and maintains its own in-memory copy of the namespace, which is always in sync with the active NameNode namespace state. Only one Backup node may be registered with the NameNode at once.

- j. Backup node：对 Checkpoint node 的扩展。除了进行 checkpoint 之外，它还接收来自 NameNode 的 edits 流，并维护自己的名称空间的内存副本，该副本始终与活跃的 NameNode 名称空间状态同步。NameNode 一次只能注册一个 Backup node。

## 3、Prerequisites

> The following documents describe how to install and set up a Hadoop cluster:

以下文档介绍了 Hadoop 集群的安装和搭建方法:

- [Single Node Setup](https://hadoop.apache.org/docs/r3.2.1/hadoop-project-dist/hadoop-common/SingleCluster.html) for first-time users.
- [Cluster Setup](https://hadoop.apache.org/docs/r3.2.1/hadoop-project-dist/hadoop-common/ClusterSetup.html) for large, distributed clusters.

> The rest of this document assumes the user is able to set up and run a HDFS with at least one DataNode. For the purpose of this document, both the NameNode and DataNode could be running on the same physical machine.

本文的其余部分假设用户能够设置和运行至少具有一个 DataNode 的 HDFS。

在本文中，NameNode 和 DataNode 可以运行在同一台物理机器上。

## 4、Web Interface

> NameNode and DataNode each run an internal web server in order to display basic information about the current status of the cluster. With the default configuration, the NameNode front page is at http://namenode-name:9870/. It lists the DataNodes in the cluster and basic statistics of the cluster. The web interface can also be used to browse the file system (using “Browse the file system” link on the NameNode front page).

NameNode 和 DataNode 都运行一个内部的 web 服务，以显示集群当前状态的基本信息。

使用默认配置，NameNode 的首页在 http://namenode-name:9870/。

它列出集群中的 DataNodes 和集群的基本统计信息。web 界面也可以用来浏览文件系统(使用 NameNode 首页上的 “browse The file system” 链接)。

## 5、Shell Commands

> Hadoop includes various shell-like commands that directly interact with HDFS and other file systems that Hadoop supports. The command bin/hdfs dfs -help lists the commands supported by Hadoop shell. Furthermore, the command bin/hdfs dfs -help command-name displays more detailed help for a command. These commands support most of the normal files system operations like copying files, changing file permissions, etc. It also supports a few HDFS specific operations like changing replication of files. For more information see [File System Shell Guide](https://hadoop.apache.org/docs/r3.2.1/hadoop-project-dist/hadoop-common/FileSystemShell.html).

Hadoop 包括各种类似 shell 的命令，它们直接与 HDFS 和 Hadoop 支持的其他文件系统交互。

`bin/hdfs dfs -help` 命令列出了 Hadoop shell 支持的命令。`bin/hdfs dfs -help command-name` 命令可以显示命令的详细帮助。

这些命令支持大多数正常的文件系统操作，如复制文件、更改文件权限等。它还支持一些 HDFS 特定的操作，如更改文件复制。有关更多信息，请参见文件系统Shell指南。

### 5.1、DFSAdmin Command

> The bin/hdfs dfsadmin command supports a few HDFS administration related operations. The bin/hdfs dfsadmin -help command lists all the commands currently supported. For e.g.:

`bin/hdfs dfsadmin` 命令仅支持部分 HDFS 管理员相关的操作。`bin/hdfs dfsadmin -help` 列出了当前支持的所有命令。例如:

> -report: reports basic statistics of HDFS. Some of this information is also available on the NameNode front page.

- -report:报告 HDFS 的基本统计信息。其中一些信息也可以在 NameNode 的首页找到。

> -safemode: though usually not required, an administrator can manually enter or leave Safemode.

- -safemode:虽然通常不需要，但管理员可以手动进入或离开安全模式。

> -finalizeUpgrade: removes previous backup of the cluster made during last upgrade.

- -finalizeUpgrade:删除上次升级时集群的备份。

> -refreshNodes: Updates the namenode with the set of datanodes allowed to connect to the namenode. By default, Namenodes re-read datanode hostnames in the file defined by dfs.hosts, dfs.hosts.exclude Hosts defined in dfs.hosts are the datanodes that are part of the cluster. If there are entries in dfs.hosts, only the hosts in it are allowed to register with the namenode. Entries in dfs.hosts.exclude are datanodes that need to be decommissioned. Alternatively if dfs.namenode.hosts.provider.classname is set to org.apache.hadoop.hdfs.server.blockmanagement.CombinedHostFileManager, all include and exclude hosts are specified in the JSON file defined by dfs.hosts. Datanodes complete decommissioning when all the replicas from them are replicated to other datanodes. Decommissioned nodes are not automatically shutdown and are not chosen for writing for new replicas.

- -refreshNodes:使用允许连接到 namenode 的 datanode 集更新 namenode。

	默认情况下，Namenodes 会重读 `dfs.hosts` 和 `dfs.hosts.exclude` 中定义的文件中的 datanode 主机名。

	`dfs.hosts` 中定义的主机是属于集群的 datanode。如果在 `dfs.hosts` 中有条目，只有其中的主机才被允许向 namenode 注册。

	在 `dfs.hosts.exclude` 中的条目是需要注销的 datanode。

	或者如果 `dfs.namenode.hosts.provider.classname` 设置为 `org.apache.hadoop.hdfs.server.blockmanagement.CombinedHostFileManager` 。所有包含的主机和排除的主机都在 `dfs.hosts` 定义的 JSON 文件中指定。

	当所有的副本都被复制到其他 datanode 时，datanode 就完成了注销。注销节点不会自动关闭，也不会被选择用于写入新的副本。

> -printTopology : Print the topology of the cluster. Display a tree of racks and datanodes attached to the tracks as viewed by the NameNode.

- - printopology:打印集群的拓扑。。。。

> For command usage, see [dfsadmin](https://hadoop.apache.org/docs/r3.2.1/hadoop-project-dist/hadoop-hdfs/HDFSCommands.html#dfsadmin).

## 6、Secondary NameNode

> The NameNode stores modifications to the file system as a log appended to a native file system file, edits. When a NameNode starts up, it reads HDFS state from an image file, fsimage, and then applies edits from the edits log file. It then writes new HDFS state to the fsimage and starts normal operation with an empty edits file. Since NameNode merges fsimage and edits files only during start up, the edits log file could get very large over time on a busy cluster. Another side effect of a larger edits file is that next restart of NameNode takes longer.

NameNode 存储对文件系统的修改，以一条日志的形式追加到一个原生文件系统文件 edits。

当 NameNode 启动时，它从 fsimage 映像文件中读取 HDFS 状态，并应用 edits 日志文件中的修改。然后它将新的 HDFS 状态写到 fsimage，并使用一个空的 edits 文件开始常规操作。 

因为 NameNode 在启动期间合并 fsimage 和 edits 文件，在一个繁忙的集群中，edits 日志文件会随时间变得非常大。一个大的 edits 文件的负面作用就是在 NameNode 下次重启时会花费更长的时间。

> The secondary NameNode merges the fsimage and the edits log files periodically and keeps edits log size within a limit. It is usually run on a different machine than the primary NameNode since its memory requirements are on the same order as the primary NameNode.

secondary NameNode 周期合并 fsimage 和 edits 日志文件，保持 edits 日志大小在一个限制范围内。它通常运行在一个与主 NameNode 不同的机器上，因为它的内存需求与主 NameNode 的 order 相同。

> The start of the checkpoint process on the secondary NameNode is controlled by two configuration parameters.

在 secondary NameNode 启动的 checkpoint 进程被如下两个配置参数控制：

> `dfs.namenode.checkpoint.period`, set to 1 hour by default, specifies the maximum delay between two consecutive checkpoints, and

- `dfs.namenode.checkpoint.period`：默认设为 1 小时，指定了在两次连续 checkpoints 见的最大的延迟。

> `dfs.namenode.checkpoint.txns`, set to 1 million by default, defines the number of uncheckpointed transactions on the NameNode which will force an urgent checkpoint, even if the checkpoint period has not been reached.

- `dfs.namenode.checkpoint.txns`：默认设为 1 百万，定义了 NameNode 上 uncheckpointed 事务的数量，这些事务将强制设置紧急 checkpoint，即使尚未达到 checkpoint 周期。

> The secondary NameNode stores the latest checkpoint in a directory which is structured the same way as the primary NameNode’s directory. So that the check pointed image is always ready to be read by the primary NameNode if necessary.

secondary NameNode 在和主 NameNode 相同的目录结构下存储最新的 checkpoint。为了在必要情况下，check pointed 映像总是准备被主 NameNode 读取。

> For command usage, see [secondarynamenode](https://hadoop.apache.org/docs/r3.2.1/hadoop-project-dist/hadoop-hdfs/HDFSCommands.html#secondarynamenode).

## 7、Checkpoint Node

> NameNode persists its namespace using two files: fsimage, which is the latest checkpoint of the namespace and edits, a journal (log) of changes to the namespace since the checkpoint. When a NameNode starts up, it merges the fsimage and edits journal to provide an up-to-date view of the file system metadata. The NameNode then overwrites fsimage with the new HDFS state and begins a new edits journal.

NameNode 使用两个文件来持久化它的命名空间：

- fsimage：命名空间的最新的 checkpoint
- edits：自 checkpoint 以来命名空间更改的日志

当 NameNode 启动时，它合并 fsimage 和 edits 日志，来提高文件系统元数据的最新的视图。然后，NameNode使用新的 HDFS 状态覆盖 fsimage，并开始一个新的 edits 日志。

> The Checkpoint node periodically creates checkpoints of the namespace. It downloads fsimage and edits from the active NameNode, merges them locally, and uploads the new image back to the active NameNode. The Checkpoint node usually runs on a different machine than the NameNode since its memory requirements are on the same order as the NameNode. The Checkpoint node is started by `bin/hdfs namenode -checkpoint` on the node specified in the configuration file.

Checkpoint 节点周期地创建命名空间的 checkpoints。它从活跃的 NameNode 下载 fsimage 和 edits，在本地合并它们，然后上传新的映像到活跃的 NameNode。

Checkpoint 节点通常运行在一个与主 NameNode 不同的机器上，因为它的内存需求与主 NameNode 的 order 相同。

Checkpoint 节点通过在配置文件中指定的节点上指定 `bin/hdfs namenode -checkpoint` 来启动。

> The location of the Checkpoint (or Backup) node and its accompanying web interface are configured via the `dfs.namenode.backup.address` and `dfs.namenode.backup.http-address` configuration variables.

Checkpoint （或 Backup）节点的位置及其 web 界面通过 `dfs.namenode.backup.address` 和 `dfs.namenode.backup.http-address` 配置。

> The start of the checkpoint process on the Checkpoint node is controlled by two configuration parameters.

Checkpoint 节点上的 checkpoint 进程的启动由下面两个参数控制：

> `dfs.namenode.checkpoint.period`, set to 1 hour by default, specifies the maximum delay between two consecutive checkpoints

- `dfs.namenode.checkpoint.period`：默认设为 1 小时，指定了在两次连续 checkpoints 见的最大的延迟。

> `dfs.namenode.checkpoint.txns`, set to 1 million by default, defines the number of uncheckpointed transactions on the NameNode which will force an urgent checkpoint, even if the checkpoint period has not been reached.

- `dfs.namenode.checkpoint.txns`：默认设为 1 百万，定义了 NameNode 上 uncheckpointed 事务的数量，这些事务将强制设置紧急 checkpoint，即使尚未达到 checkpoint 周期。

> The Checkpoint node stores the latest checkpoint in a directory that is structured the same as the NameNode’s directory. This allows the checkpointed image to be always available for reading by the NameNode if necessary. See Import checkpoint.

Checkpoint 节点在和主 NameNode 相同的目录结构下存储最新的 checkpoint。这就允许在必要情况下，checkpointed 映像总是准备被 NameNode 读取。见 Import checkpoint。

> Multiple checkpoint nodes may be specified in the cluster configuration file.

> For command usage, see [namenode](https://hadoop.apache.org/docs/r3.2.1/hadoop-project-dist/hadoop-hdfs/HDFSCommands.html#namenode).

## 8、Backup Node

> The Backup node provides the same checkpointing functionality as the Checkpoint node, as well as maintaining an in-memory, up-to-date copy of the file system namespace that is always synchronized with the active NameNode state. Along with accepting a journal stream of file system edits from the NameNode and persisting this to disk, the Backup node also applies those edits into its own copy of the namespace in memory, thus creating a backup of the namespace.

Backup 节点提供了和 Checkpoint 节点相同的 checkpoint 功能，也维持了一个内存中的、最新的文件系统命名空间的副本，这个副本总是和活跃的 NameNode 状态的保持同步。

除了接受来自 NameNode 的文件系统 edits 的日志流，并将其保存到磁盘之外，Backup 节点还将这些 edits 应用到内存中的自己的命名空间副本中，从而创建命名空间的备份。

> The Backup node does not need to download fsimage and edits files from the active NameNode in order to create a checkpoint, as would be required with a Checkpoint node or Secondary NameNode, since it already has an up-to-date state of the namespace state in memory. The Backup node checkpoint process is more efficient as it only needs to save the namespace into the local fsimage file and reset edits.

Backup 节点不需要从活跃的 NameNode 下载 fsimage 和 edits 文件，来创建一个 checkpoint，这是 Checkpoint 节点或 Secondary NameNode 所要求的，因为它早已在内存中有一个命名空间状态的最新状态。

Backup 节点的 checkpoint 过程更高效，因为它仅需要将命名空间保存到本地 fsimage 文件，并重置 edits。

> As the Backup node maintains a copy of the namespace in memory, its RAM requirements are the same as the NameNode.

当 Backup 节点在内存中维持一个命名空间的副本时，它的 RAM 要求和 NameNode 相同。 

> The NameNode supports one Backup node at a time. No Checkpoint nodes may be registered if a Backup node is in use. Using multiple Backup nodes concurrently will be supported in the future.

NameNode 一次支持一个 Backup 节点。如果使用了 Backup 节点，那么就不需要再注册 Checkpoint 节点。今后将支持多个 Backup 节点同时使用。

> The Backup node is configured in the same manner as the Checkpoint node. It is started with `bin/hdfs namenode -backup`.

Backup 节点的配置方式和 Checkpoint 节点相同。使用 `bin/hdfs namenode -backup` 启动。

> The location of the Backup (or Checkpoint) node and its accompanying web interface are configured via the `dfs.namenode.backup.address` and `dfs.namenode.backup.http-address` configuration variables.

Backup （或 Checkpoint）节点的位置及其 web 界面通过 `dfs.namenode.backup.address` 和 `dfs.namenode.backup.http-address` 配置。

> Use of a Backup node provides the option of running the NameNode with no persistent storage, delegating all responsibility for persisting the state of the namespace to the Backup node. To do this, start the NameNode with the `-importCheckpoint` option, along with specifying no persistent storage directories of type edits `dfs.namenode.edits.dir` for the NameNode configuration.

Backup 节点的使用提供了一个可以在没有持久存储的情况下运行 NameNode 的选项，将持久化命名空间状态的所有责任委托给 Backup 节点。

为此，使用 `-importCheckpoint` 选项启动 NameNode，并为 NameNode 指定不包含类型 edits 的持久存储目录 `dfs.namenode.edits.dir`。

> For a complete discussion of the motivation behind the creation of the Backup node and Checkpoint node, see [HADOOP-4539](https://issues.apache.org/jira/browse/HADOOP-4539). For command usage, see [namenode](https://hadoop.apache.org/docs/r3.2.1/hadoop-project-dist/hadoop-hdfs/HDFSCommands.html#namenode).

## 9、Import Checkpoint

> The latest checkpoint can be imported to the NameNode if all other copies of the image and the edits files are lost. In order to do that one should:

如果 image 和 edits 文件的所有其他副本都丢失了，那么最新的 checkpoint 可以导入到 NameNode。为了做到这一点，应该：

> Create an empty directory specified in the `dfs.namenode.name.dir` configuration variable;

- 创建一个 `dfs.namenode.name.dir` 中指定的空目录

> Specify the location of the checkpoint directory in the configuration variable `dfs.namenode.checkpoint.dir`;

- 在 `dfs.namenode.checkpoint.dir` 中，指定 checkpoint 目录的位置

> and start the NameNode with `-importCheckpoint` option.

- 使用 `-importCheckpoint` 选项启动 NameNode

> The NameNode will upload the checkpoint from the `dfs.namenode.checkpoint.dir` directory and then save it to the NameNode directory(s) set in `dfs.namenode.name.dir`. The NameNode will fail if a legal image is contained in `dfs.namenode.name.dir`. The NameNode verifies that the image in `dfs.namenode.checkpoint.dir` is consistent, but does not modify it in any way.

NameNode 将从 `dfs.namenode.checkpoint.dir` 目录上传 checkpoint，然后将其保存到 `dfs.namenode.name.dir` 设置的 NameNode 目录。

如果合法的映像包含在 `dfs.namenode.name.dir` 中，那么 NameNode 将失败。NameNode 会验证 `dfs.namenode.checkpoint.dir` 中的映像是否一致，但不会以任何方式修改它。

> For command usage, see [namenode](https://hadoop.apache.org/docs/r3.2.1/hadoop-project-dist/hadoop-hdfs/HDFSCommands.html#namenode).

## 10、Balancer

> HDFS data might not always be be placed uniformly across the DataNode. One common reason is addition of new DataNodes to an existing cluster. While placing new blocks (data for a file is stored as a series of blocks), NameNode considers various parameters before choosing the DataNodes to receive these blocks. Some of the considerations are:

HDFS 数据可能不是均匀的放置在 DataNode 上。一个常见的原因是在现有的集群中添加了新的 DataNodes。

在放置新块（文件的数据存储为一系列块）时，NameNode 在选择接收这些块的 DataNodes 之前要考虑各种参数。

需要考虑的事项有：

> Policy to keep one of the replicas of a block on the same node as the node that is writing the block.

- 策略，将块的其中一个副本与写入块的节点保持在同一个节点上。

> Need to spread different replicas of a block across the racks so that cluster can survive loss of whole rack.

- 需要在机架上散布块的不同副本，以便集群在丢失整个机架时能够存活。

> One of the replicas is usually placed on the same rack as the node writing to the file so that cross-rack network I/O is reduced.

- 其中一个副本通常与写入文件的节点放在同一个机架上，这样就减少了跨机架的网络 I/O。

> Spread HDFS data uniformly across the DataNodes in the cluster.

- 将 HDFS 数据均匀地分布在集群内的 DataNodes 节点上。

> Due to multiple competing considerations, data might not be uniformly placed across the DataNodes. HDFS provides a tool for administrators that analyzes block placement and rebalanaces data across the DataNode. A brief administrator’s guide for balancer is available at [HADOOP-1652](https://issues.apache.org/jira/browse/HADOOP-1652).

出于多种竞争考虑，数据可能不会均匀地放置在各个 DataNodes 中。HDFS 为管理员提供了一个分析块位置和跨 DataNode 重新平衡数据的工具。

> For command usage, see [balancer](https://hadoop.apache.org/docs/r3.2.1/hadoop-project-dist/hadoop-hdfs/HDFSCommands.html#balancer).

## 11、Rack Awareness

> A HDFS cluster can recognize the topology of racks where each nodes are put. It is important to configure this topology in order to optimize the data capacity and usage. For more detail, please check the [rack awareness](https://hadoop.apache.org/docs/r3.2.1/hadoop-project-dist/hadoop-common/RackAwareness.html) in common document.

HDFS 集群可以识别每个节点所在机架的拓扑结构。为了优化数据容量和使用，配置此拓扑非常重要。

## 12、Safemode

> During start up the NameNode loads the file system state from the fsimage and the edits log file. It then waits for DataNodes to report their blocks so that it does not prematurely start replicating the blocks though enough replicas already exist in the cluster. During this time NameNode stays in Safemode. Safemode for the NameNode is essentially a read-only mode for the HDFS cluster, where it does not allow any modifications to file system or blocks. Normally the NameNode leaves Safemode automatically after the DataNodes have reported that most file system blocks are available. If required, HDFS could be placed in Safemode explicitly using `bin/hdfs dfsadmin -safemode` command. NameNode front page shows whether Safemode is on or off. A more detailed description and configuration is maintained as JavaDoc for `setSafeMode()`.

在启动期间，NameNode 从 fsimage 和 edits 日志文件加载文件系统状态。然后等待 DataNodes 报告它们的块，这样它就不会过早地开始复制这些块，尽管集群中已经有足够的副本。

在此期间 NameNode 处于 Safemode 状态。NameNode 的 Safemode 本质上是 HDFS 集群的只读模式，不允许对文件系统或块进行任何修改。

通常在 DataNodes 向文件系统报告块可用后，NameNode 会自动离开 Safemode。

如果需要，可以使用 `bin/hdfs dfsadmin -safemode` 命令显式地将 HDFS 放置在 Safemode 中。NameNode 首页显示 Safemode 是开启还是关闭。

## 13、fsck

> HDFS supports the fsck command to check for various inconsistencies. It is designed for reporting problems with various files, for example, missing blocks for a file or under-replicated blocks. Unlike a traditional fsck utility for native file systems, this command does not correct the errors it detects. Normally NameNode automatically corrects most of the recoverable failures. By default fsck ignores open files but provides an option to select all files during reporting. The HDFS fsck command is not a Hadoop shell command. It can be run as bin/hdfs fsck. For command usage, see [fsck](https://hadoop.apache.org/docs/r3.2.1/hadoop-project-dist/hadoop-hdfs/HDFSCommands.html#fsck). fsck can be run on the whole file system or on a subset of files.

HDFS 支持 fsck 命令来检查各种不一致。

它是为报告各种文件的问题而设计的，例如，某个文件缺失块或副本不足的块。

与用于本地文件系统的传统 fsck 实用程序不同，这个命令不会纠正它检测到的错误。通常 NameNode 会自动纠正大部分可恢复故障。

默认情况下，fsck 忽略打开的文件，但在报告过程中提供了选择所有文件的选项。HDFS 的 fsck 命令不是 Hadoop shell 命令。可以以 `bin/hdfs fsck` 的形式运行。

fsck 可以在整个文件系统上运行，也可以在文件的一个子集上运行。

## 14、fetchdt

> HDFS supports the fetchdt command to fetch Delegation Token and store it in a file on the local system. This token can be later used to access secure server (NameNode for example) from a non secure client. Utility uses either RPC or HTTPS (over Kerberos) to get the token, and thus requires kerberos tickets to be present before the run (run kinit to get the tickets). The HDFS fetchdt command is not a Hadoop shell command. It can be run as `bin/hdfs fetchdt DTfile`. After you got the token you can run an HDFS command without having Kerberos tickets, by pointing `HADOOP_TOKEN_FILE_LOCATION` environmental variable to the delegation token file. For command usage, see [fetchdt](https://hadoop.apache.org/docs/r3.2.1/hadoop-project-dist/hadoop-hdfs/HDFSCommands.html#fetchdt) command.

HDFS 支持通过 fetchdt 命令获取委托令牌，并将其存储在本地系统的文件中。

稍后可以使用此令牌从非安全客户端访问安全服务器（例如NameNode）。实用程序使用 RPC 或 HTTPS（通过Kerberos）来获得令牌，因此需要在运行之前提供 Kerberos 票据(运行 kinit 来获得票据)。

HDFS 的 fetchdt 命令不是 Hadoop shell 命令。可以以 `bin/hdfs fetchdt DTfile` 的形式运行。在获得令牌之后，你可以在没有 Kerberos 票据的情况下运行 HDFS 命令，方法是将 `HADOOP_TOKEN_FILE_LOCATION` 环境变量指向委托令牌文件。

## 15、Recovery Mode

> Typically, you will configure multiple metadata storage locations. Then, if one storage location is corrupt, you can read the metadata from one of the other storage locations.

通常，你将配置多个元数据存储位置。然后，如果一个存储位置损坏，则可以从其他存储位置读取元数据。

> However, what can you do if the only storage locations available are corrupt? In this case, there is a special NameNode startup mode called Recovery mode that may allow you to recover most of your data.

但是，如果仅有的可用存储位置被破坏了，你该怎么办呢？在这种情况下，有一种特殊的 NameNode 启动模式称为恢复模式，它可以允许你恢复大部分数据。

> You can start the NameNode in recovery mode like so: `namenode -recover`

以恢复模式启动 NameNode: `namenode -recover`

> When in recovery mode, the NameNode will interactively prompt you at the command line about possible courses of action you can take to recover your data.

当处于恢复模式时，NameNode 将在命令行上交互式地提示你可以采取哪些措施来恢复数据。

> If you don’t want to be prompted, you can give the `-force` option. This option will force recovery mode to always select the first choice. Normally, this will be the most reasonable choice.

如果你不想被提示，你可以给出 `-force` 选项。此选项将强制恢复模式始终选择第一个选项。通常，这将是最合理的选择。

> Because Recovery mode can cause you to lose data, you should always back up your edit log and fsimage before using it.

因为恢复模式可能会导致数据丢失，所以在使用 edit 日志和 fsimage 之前，应该始终备份它。

## 16、Upgrade and Rollback

> When Hadoop is upgraded on an existing cluster, as with any software upgrade, it is possible there are new bugs or incompatible changes that affect existing applications and were not discovered earlier. In any non-trivial HDFS installation, it is not an option to loose any data, let alone to restart HDFS from scratch. HDFS allows administrators to go back to earlier version of Hadoop and rollback the cluster to the state it was in before the upgrade. HDFS upgrade is described in more detail in [Hadoop Upgrade](http://wiki.apache.org/hadoop/Hadoop_Upgrade) Wiki page. HDFS can have one such backup at a time. Before upgrading, administrators need to remove existing backup using `bin/hadoop dfsadmin -finalizeUpgrade` command. The following briefly describes the typical upgrade procedure:

当 Hadoop 在现有集群上升级时，就像任何软件升级一样，可能会有新的 bug 或不兼容的更改影响现有的应用程序，而之前没有发现。

在任何重要的 HDFS 安装中，都不能丢失任何数据，更不用说从头重新启动 HDFS 了。

HDFS 允许管理员返回到 Hadoop 的早期版本，并将集群回滚到升级前的状态。

HDFS 一次只能有一个这样的备份。升级前，管理员需要使用 `bin/hadoop dfsadmin -finalizeUpgrade` 命令删除已有备份。典型升级流程如下：

> Before upgrading Hadoop software, finalize if there an existing backup.

- 在升级 Hadoop 软件之前，确定是否存在现有的备份。

> Stop the cluster and distribute new version of Hadoop.

- 停止集群，并分发新版本的 Hadoop。

> Run the new version with `-upgrade` option (`sbin/start-dfs.sh -upgrade`).

- 运行带有 `-upgrade` 选项的新版本(`sbin/start-dfs.sh -upgrade`)。

> Most of the time, cluster works just fine. Once the new HDFS is considered working well (may be after a few days of operation), finalize the upgrade. Note that until the cluster is finalized, deleting the files that existed before the upgrade does not free up real disk space on the DataNodes.

- 大多数情况下，集群工作得很好。一旦新的 HDFS 被认为工作良好(可能在几天的操作之后)，最后完成升级。注意，在集群完成之前，删除升级前存在的文件不会释放 DataNodes 上的实际磁盘空间。

> If there is a need to move back to the old version,

- 如果需要回到旧版本，

	- 停止集群，分发先前的 Hadoop 版本。

	- 在 namenode 上执行 rollback 命令(`bin/hdfs namenode -rollback`)。

	- 使用 rollback 选项启动集群。(`sbin/start-dfs.sh -rollback`)。

> stop the cluster and distribute earlier version of Hadoop.

> run the rollback command on the namenode (`bin/hdfs namenode -rollback`).

> start the cluster with rollback option. (`sbin/start-dfs.sh -rollback`).

> When upgrading to a new version of HDFS, it is necessary to rename or delete any paths that are reserved in the new version of HDFS. If the NameNode encounters a reserved path during upgrade, it will print an error like the following:

升级到新的 HDFS 版本时，需要重命名或删除 HDFS 新版本中保留的任何路径。如果 NameNode 在升级过程中遇到了保留路径，会打印如下错误信息:

	`/.reserved` is a reserved path and .snapshot is a reserved path component in this version of HDFS. Please rollback and delete or rename this path, or upgrade with the -renameReserved [key-value pairs] option to automatically rename these paths during upgrade.

> Specifying `-upgrade -renameReserved [optional key-value pairs]` causes the NameNode to automatically rename any reserved paths found during startup. For example, to rename all paths named `.snapshot` to `.my-snapshot` and `.reserved` to `.my-reserved`, a user would specify `-upgrade -renameReserved .snapshot=.my-snapshot`,`.reserved=.my-reserved`.

指定 `-upgrade -renameReserved [optional key-value pairs]` 将导致 NameNode 自动地重命名启动过程中发现的任意保留路径。

例如，要将所有名为 `.snapshot` 的路径重命名为 `.my-snapshot`，将 `.reserved` 路径重命名为 `.my-reserved`，用户需要指定 `-upgrade -renameReserved .snapshot=.my-snapshot`,`.reserved=.my-reserved`。

> If no key-value pairs are specified with `-renameReserved`, the NameNode will then suffix reserved paths with `.<LAYOUT-VERSION>.UPGRADE_RENAMED`, e.g. `.snapshot.-51.UPGRADE_RENAMED`.

如果没有使用 `-renameReserved` 指定键值对，则 NameNode 将用 `.<LAYOUT-VERSION>.UPGRADE_RENAMED` 作为保留路径的后缀。例如 `.snapshot.-51.UPGRADE_RENAMED`。

> There are some caveats to this renaming process. It’s recommended, if possible, to first `hdfs dfsadmin -saveNamespace` before upgrading. This is because data inconsistency can result if an edit log operation refers to the destination of an automatically renamed file.

对于这个重命名过程有一些注意事项。如果可能，建议在升级前先使用 `hdfs dfsadmin -saveNamespace`。这是因为如果 edit 日志操作指向自动重命名文件的目标，就会导致数据不一致。

## 17、DataNode Hot Swap Drive

> Datanode supports hot swappable drives. The user can add or replace HDFS data volumes without shutting down the DataNode. The following briefly describes the typical hot swapping drive procedure:

Datanode 支持热插拔。用户可以在不关闭 DataNode 的情况下增加或替换 HDFS 数据卷。以下简要介绍了典型的热插拔过程：

> If there are new storage directories, the user should format them and mount them appropriately.

- 如果有新的存储目录，用户应该格式化它们，并适当地挂载它们。

> The user updates the DataNode configuration `dfs.datanode.data.dir` to reflect the data volume directories that will be actively in use.

- 用户更新 DataNode 配置 `dfs.datanode.data.dir` ，来反映将要被积极地使用的数据卷目录。

> The user runs `dfsadmin -reconfig datanode HOST:PORT start` to start the reconfiguration process. The user can use `dfsadmin -reconfig datanode HOST:PORT status` to query the running status of the reconfiguration task.

- 执行 `dfsadmin -reconfig datanode HOST:PORT start` 命令，启动重新配置进程。用户可以使用 `dfsadmin -reconfig datanode HOST:PORT status` 查询重新配置任务的运行状态。

> Once the reconfiguration task has completed, the user can safely `umount` the removed data volume directories and physically remove the disks.

- 一旦重新配置任务完成，用户就可以安全地挂载删除的数据卷目录，并从物理上删除磁盘。

## 18、File Permissions and Security

> The file permissions are designed to be similar to file permissions on other familiar platforms like Linux. Currently, security is limited to simple file permissions. The user that starts NameNode is treated as the superuser for HDFS. Future versions of HDFS will support network authentication protocols like Kerberos for user authentication and encryption of data transfers. The details are discussed in the Permissions Guide.

文件权限被设计成类似于 Linux 等其他熟悉平台上的文件权限。

目前，安全性仅限于简单文件权限。启动 NameNode 的用户被视为 HDFS 的超级用户。HDFS 的未来版本将支持 Kerberos 等网络认证协议，用于用户认证和数据传输的加密。

## 19、Scalability

> Hadoop currently runs on clusters with thousands of nodes. The [PoweredBy](http://wiki.apache.org/hadoop/PoweredBy) Wiki page lists some of the organizations that deploy Hadoop on large clusters. HDFS has one NameNode for each cluster. Currently the total memory available on NameNode is the primary scalability limitation. On very large clusters, increasing average size of files stored in HDFS helps with increasing cluster size without increasing memory requirements on NameNode. The default configuration may not suite very large clusters. The [FAQ](http://wiki.apache.org/hadoop/FAQ) Wiki page lists suggested configuration improvements for large Hadoop clusters.

Hadoop 目前运行在拥有数千个节点的集群上。PoweredBy Wiki 页面列出了一些在大型集群上部署 Hadoop 的组织。

HDFS 对于每个集群都有一个 NameNode。目前，NameNode 上可用的总内存是主要的可伸缩性限制。

在非常大的集群上，增加存储在 HDFS 中的文件的平均大小有助于增加集群大小，而不会增加对 NameNode 的内存需求。默认配置可能不适合非常大的集群。FAQ Wiki 页面列出了针对大型 Hadoop 集群的配置改进建议。

## 20、Related Documentation

This user guide is a good starting point for working with HDFS. While the user guide continues to improve, there is a large wealth of documentation about Hadoop and HDFS. The following list is a starting point for further exploration:

- [Hadoop Site](http://hadoop.apache.org/): The home page for the Apache Hadoop site.
- [Hadoop Wiki](http://wiki.apache.org/hadoop/FrontPage): The home page (FrontPage) for the Hadoop Wiki. Unlike the released documentation, which is part of Hadoop source tree, Hadoop Wiki is regularly edited by Hadoop Community.
- [FAQ](http://wiki.apache.org/hadoop/FAQ): The FAQ Wiki page.
- [Hadoop JavaDoc API](https://hadoop.apache.org/docs/r3.2.1/api/index.html).
- Hadoop User Mailing List: user[at]hadoop.apache.org.
- [Explore hdfs-default.xml](https://hadoop.apache.org/docs/r3.2.1/hadoop-project-dist/hadoop-hdfs/hdfs-default.xml). It includes brief description of most of the configuration variables available.
- [HDFS Commands Guide](https://hadoop.apache.org/docs/r3.2.1/hadoop-project-dist/hadoop-hdfs/HDFSCommands.html): HDFS commands usage.