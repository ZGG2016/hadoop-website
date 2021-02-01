# Hadoop Benchmarking

[TOC]

> This page is to discuss benchmarking Hadoop using tools it provides.

本页面将讨论使用 Hadoop 提供的工具对 Hadoop 进行基准测试。

## 1、NNThroughputBenchmark

### 1.1、Overview

> NNThroughputBenchmark, as its name indicates, is a name-node throughput benchmark, which runs a series of client threads on a single node against a name-node. If no name-node is configured, it will firstly start a name-node in the same process (standalone mode), in which case each client repetitively performs the same operation by directly calling the respective name-node methods. Otherwise, the benchmark will perform the operations against a remote name-node via client protocol RPCs (remote mode). Either way, all clients are running locally in a single process rather than remotely across different nodes. The reason is to avoid communication overhead caused by RPC connections and serialization, and thus reveal the upper bound of pure name-node performance.

NNThroughputBenchmark 是一个 name-node 吞吐量基准测试，它在一个 name-node 节点上运行一系列客户端线程。

如果没有配置 name-node，它将首先在同一进程中启动一个 name-node(独立模式)，在这种情况下，每个客户端通过直接调用各自的 name-node 方法重复执行相同的操作。

否则，基准测试将通过客户端协议 RPCs(远程模式)对远程 name-node 执行操作。

无论采用哪种方式，所有客户端都在单个进程中本地运行，而不是在不同节点之间远程运行。其原因是为了避免 RPC 连接和序列化导致的通信开销，从而揭示纯 name-node 性能的上限。

> The benchmark first generates inputs for each thread so that the input generation overhead does not effect the resulting statistics. The number of operations performed by threads is practically the same. Precisely, the difference between the number of operations performed by any two threads does not exceed 1. Then the benchmark executes the specified number of operations using the specified number of threads and outputs the resulting stats by measuring the number of operations performed by the name-node per second.

基准测试首先为每个线程生成输入，以便输入生成开销不会影响结果统计数据。

线程执行的操作数实际上是相同的。准确地说，任意两个线程执行的操作数之差不超过1。

然后，基准测试使用指定线程数执行指定的操作数，并通过测量 name-node 每秒执行的操作数量输出结果统计信息。

### 1.2、Commands

> The general command line syntax is:

语法：

	hadoop org.apache.hadoop.hdfs.server.namenode.NNThroughputBenchmark [genericOptions] [commandOptions]

#### 1.2.1、Generic Options

> This benchmark honors the [Hadoop command-line Generic Options](https://hadoop.apache.org/docs/r3.2.1/hadoop-project-dist/hadoop-common/CommandsManual.html#Generic_Options) to alter its behavior. The benchmark, as other tools, will rely on the fs.defaultFS config, which is overridable by -fs command option, to run standalone mode or remote mode. If the fs.defaultFS scheme is not specified or is file (local), the benchmark will run in standalone mode. Specially, the remote name-node config dfs.namenode.fs-limits.min-block-size should be set as 16 while in standalone mode the benchmark turns off minimum block size verification for its internal name-node.

这个基准测试使用 Hadoop 命令行通用选项来改变其行为。

基准测试和其他工具一样，将依赖于 fs.defaultFS 配置，该配置可被 `-fs` 命令选项覆盖，以运行独立模式或远程模式。

如果未指定 `fs.defaultFS` 方案，或者是 `file` (本地)，基准测试将以独立模式运行。

特别是，远程 name-node 配置 `dfs.namenode.fs-limits.min-block-size` 应该设置为16，而在独立模式下，基准测试关闭了对其内部 name-node 的最小块大小验证。

#### 1.2.2、Command Options

> The following are all supported command options:

下面是支持的命令选项：

COMMAND_OPTION        |  Description
---|:---
-op	                  |  Specify the operation. This option must be provided and should be the first option.【指定操作。必须提供这个选项，应该是第一个选项】
-logLevel             |  Specify the logging level when the benchmark runs. The default logging level is ERROR.【当运行基准测试时，指定日志级别。默认是ERROR】
-UGCacheRefreshCount  |	 After every specified number of operations, the benchmark purges the name-node’s user group cache. By default the refresh is never called.【在执行指定数量的操作后，基准测试将清除name-node的用户组缓存。默认情况下，不会调用刷新。】
-keepResults	      |  If specified, do not clean up the name-space after execution. By default the name-space will be removed after test.【如果指定了，在执行后，不会清理名称空间。默认，测试之后，名称空间将被移除】

> Operations Supported
> Following are all the operations supported along with their respective operation-specific parameters (all optional) and default values.

支持的操作

下面是支持的所有操作，以及它们各自特定于操作的参数(所有可选)和默认值。

OPERATION_OPTION   |   Operation-specific parameters
---|:---
all	               |   options for other operations
create	           |   [-threads 3] [-files 10] [-filesPerDir 4] [-close]
mkdirs	           |   [-threads 3] [-dirs 10] [-dirsPerDir 2]
open	           |   [-threads 3] [-files 10] [-filesPerDir 4] [-useExisting]
delete	           |   [-threads 3] [-files 10] [-filesPerDir 4] [-useExisting]
fileStatus	       |   [-threads 3] [-files 10] [-filesPerDir 4] [-useExisting]
rename	           |   [-threads 3] [-files 10] [-filesPerDir 4] [-useExisting]
blockReport	       |   [-datanodes 10] [-reports 30] [-blocksPerReport 100] [-blocksPerFile 10]
replication 	   |   [-datanodes 10] [-nodesToDecommission 1] [-nodeReplicationLimit 100] [-totalBlocks 100] [-replication 3]
clean	           |   N/A


> Operation Options
> When running benchmarks with the above operation(s), please provide operation-specific parameters illustrated as following.

操作选项

当运行的基准测试带有如上的操作，请提供下面描述的参数：

OPERATION_SPECIFIC_OPTION  |  Description
---|:---
-threads	               |  Number of total threads to run the respective operation.
-files	                   |  Number of total files for the respective operation.
-dirs	                   |  Number of total directories for the respective operation.
-filesPerDir	           |  Number of files per directory.
-close	                   |  Close the files after creation.
-dirsPerDir	               |  Number of directories per directory.
-useExisting	           |  If specified, do not recreate the name-space, use existing data.
-datanodes	               |  Total number of simulated data-nodes.
-reports	               |  Total number of block reports to send.
-blocksPerReport	       |  Number of blocks per report.
-blocksPerFile	           |  Number of blocks per file.
-nodesToDecommission	   |  Total number of simulated data-nodes to decommission.
-nodeReplicationLimit	   |  The maximum number of outgoing replication streams for a data-node.
-totalBlocks	           |  Number of total blocks to operate.
-replication	           |  Replication factor. Will be adjusted to number of data-nodes if it is larger than that.

### 1.3、Reports

> The benchmark measures the number of operations performed by the name-node per second. Specifically, for each operation tested, it reports the total running time in seconds (Elapsed Time), operation throughput (Ops per sec), and average time for the operations (Average Time). The higher, the better.

基准测试测量 name-node 每秒执行的操作数。

具体来说，对于测试的每个操作，它报告总运行时间(逝去时间)以秒为单位的、操作吞吐量(每秒Ops)和操作的平均时间(平均时间)。值越高越好。

> Following is a sample reports by running following commands that opens 100K files with 1K threads against a remote name-node. See [HDFS scalability: the limits to growth](https://www.usenix.org/legacy/publications/login/2010-04/openpdfs/shvachko.pdf) for real-world benchmark stats.

下面是一个示例报告，运行以下命令，针对远程 name-node 打开 100K 个文件，其中包含 1K 个线程。

	$ hadoop org.apache.hadoop.hdfs.server.namenode.NNThroughputBenchmark -fs hdfs://nameservice:9000 -op open -threads 1000 -files 100000

	--- open inputs ---
	nrFiles = 100000
	nrThreads = 1000
	nrFilesPerDir = 4
	--- open stats  ---
	# operations: 100000
	Elapsed Time: 9510
	 Ops per sec: 10515.247108307045
	Average Time: 90