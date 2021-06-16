# HDFS Commands Guide

[TOC]

## 1、Overview

> All HDFS commands are invoked by the bin/hdfs script. Running the hdfs script without any arguments prints the description for all commands.

所有 HDFS 命令都是由 `bin/hdfs` 脚本调用。不加参数的运行 hdfs 脚本会打印所有命令的描述信息。

Usage: 

    hdfs [SHELL_OPTIONS] COMMAND [GENERIC_OPTIONS] [COMMAND_OPTIONS]

Hadoop 有一个选项解析框架，它使用解析通用选项和运行类。

> Hadoop has an option parsing framework that employs parsing generic options as well as running classes.

COMMAND_OPTIONS  |  Description
---|:---
SHELL_OPTIONS  |	The common set of shell options. These are documented on the [Commands Manual page](https://hadoop.apache.org/docs/r3.2.1/hadoop-project-dist/hadoop-common/CommandsManual.html#Shell_Options).【shell选项的命令集】
GENERIC_OPTIONS |  The common set of options supported by multiple commands. See the Hadoop [Commands Manual](https://hadoop.apache.org/docs/r3.2.1/hadoop-project-dist/hadoop-common/CommandsManual.html#Generic_Options) for more information.【多个命令支持的常见选项集。】
COMMAND COMMAND_OPTIONS  |  Various commands with their options are described in the following sections. The commands have been grouped into [User Commands](https://hadoop.apache.org/docs/r3.2.1/hadoop-project-dist/hadoop-hdfs/HDFSCommands.html#User_Commands) and [Administration Commands](https://hadoop.apache.org/docs/r3.2.1/hadoop-project-dist/hadoop-hdfs/HDFSCommands.html#Administration_Commands).【下面几节将介绍各种命令及其选项。命令被分为用户命令和管理命令】

## 2、User Commands

### classpath

Usage: 

    hdfs classpath [--glob |--jar <path> |-h |--help]

COMMAND_OPTION  | Description
---|:---
--glob          | expand wildcards
--jar path      | write classpath as manifest in jar named path
-h, --help      | print help

> Prints the class path needed to get the Hadoop jar and the required libraries. If called without arguments, then prints the classpath set up by the command scripts, which is likely to contain wildcards in the classpath entries. Additional options print the classpath after wildcard expansion or write the classpath into the manifest of a jar file. The latter is useful in environments where wildcards cannot be used and the expanded classpath exceeds the maximum supported command line length.

打印所需的类路径，以得到 hadoop jar 和所要求的库。

如果在没有参数的情况下调用，则打印由命令脚本设置的类路径，类路径条目中可能包含通配符。

其他选项在通配符展开后打印类路径，或者将类路径写入 jar 文件的清单中。后者在不能使用通配符且扩展的类路径超过了支持的最大命令行长度的环境中非常有用。


### dfs

Usage: 

    hdfs dfs [COMMAND [COMMAND_OPTIONS]]

在 hadoop 支持的文件系统上运行一个文件系统命令。

> Run a filesystem command on the file system supported in Hadoop. The various COMMAND_OPTIONS can be found at [File System Shell Guide](https://hadoop.apache.org/docs/r3.2.1/hadoop-project-dist/hadoop-common/FileSystemShell.html).

### fsck

Usage:

    hdfs fsck <path>
          [-list-corruptfileblocks |
          [-move | -delete | -openforwrite]
          [-files [-blocks [-locations | -racks | -replicaDetails | -upgradedomains]]]
          [-includeSnapshots] [-showprogress]
          [-storagepolicies] [-maintenance]
          [-blockId <blk_Id>]

COMMAND_OPTION | Description
---|:---
path           | Start checking from this path.【从这个路径开始检查】
-delete        | Delete corrupted files.【删除损坏文件】
-files         | Print out files being checked.【打印正在检查的文件。】
-files -blocks | Print out the block report【打印块报告】
-files -blocks -locations | Print out locations for every block.【打印每个块的位置】
-files -blocks -racks     | Print out network topology for data-node locations.【打印data-node位置的网络拓扑】
-files -blocks -replicaDetails |  Print out each replica details.【打印每个副本详情】
-files -blocks -upgradedomains |  Print out upgrade domains for every block.【】
-includeSnapshots | Include snapshot data if the given path indicates a snapshottable directory or there are snapshottable directories under it.【如果给定的路径指示一个可拍照目录，或者该目录下有可拍照目录，则包含拍照数据】
-list-corruptfileblocks | Print out list of missing blocks and files they belong to.【打印它们所属的缺失的块和文件】
-move  |  Move corrupted files to /lost+found.【将损坏的文件移动到/lost+found目录】
-openforwrite | Print out files opened for write.【打印待写的打开的文件】
-showprogress | Print out dots for progress in output. Default is OFF (no progress).
-storagepolicies  |  Print out storage policy summary for the blocks.【打印块的存储策略概要】
-maintenance | Print out maintenance state node details.
-blockId     | Print out information about the block.

> Runs the HDFS filesystem checking utility. See [fsck](https://hadoop.apache.org/docs/r3.2.1/hadoop-project-dist/hadoop-hdfs/HdfsUserGuide.html#fsck) for more info.

## 3、Administration Commands


### balancer

Usage:

    hdfs balancer
          [-policy <policy>]
          [-threshold <threshold>]
          [-exclude [-f <hosts-file> | <comma-separated list of hosts>]]
          [-include [-f <hosts-file> | <comma-separated list of hosts>]]
          [-source [-f <hosts-file> | <comma-separated list of hosts>]]
          [-blockpools <comma-separated list of blockpool ids>]
          [-idleiterations <idleiterations>]
          [-runDuringUpgrade]

COMMAND_OPTION | Description
---|:---
-policy `<policy>`  |  (1)datanode (default): Cluster is balanced if each datanode is balanced.【如果每个datanode是均衡的，那么集群就是均衡的】(2)blockpool: Cluster is balanced if each block pool in each datanode is balanced.【如果每个datanode中的block pool是均衡的，那么集群就是均衡的】
-threshold `<threshold>`  |  Percentage of disk capacity. This overwrites the default threshold. 【磁盘容量的百分比】
-exclude `-f <hosts-file> | <comma-separated list of hosts>`	 | Excludes the specified datanodes from being balanced by the balancer.【排除的不被均衡的节点】
-include `-f <hosts-file> | <comma-separated list of hosts>`	 | Includes only the specified datanodes to be balanced by the balancer.【被均衡的节点】
-source `-f <hosts-file> | <comma-separated list of hosts>`	 | Pick only the specified datanodes as source nodes.【仅选择指定的datanodes作为源nodes】
-blockpools `<comma-separated list of blockpool ids>`	 | The balancer will only run on blockpools included in this list.【均衡器仅会在这个列表列出的blockpools上运行】
-idleiterations `<iterations>`  | Maximum number of idle iterations before exit. This overwrites the default idleiterations(5).
-runDuringUpgrade | Whether to run the balancer during an ongoing HDFS upgrade. This is usually not desired since it will not affect used space on over-utilized machines.【是否在正在升级的HDFS中运行平衡器。这通常是不希望的，因为它不会影响过度使用的机器上的已用空间】
-h|--help	 | Display the tool usage and help information and exit.

> Runs a cluster balancing utility. An administrator can simply press Ctrl-C to stop the rebalancing process. See [Balancer](https://hadoop.apache.org/docs/r3.2.1/hadoop-project-dist/hadoop-hdfs/HdfsUserGuide.html#Balancer) for more details.

运行集群均衡工具，管理员可以 Ctrl-C 停止正均衡的进程。

blockpool 模式比 datanode 模式更严格。

> Note that the blockpool policy is more strict than the datanode policy.

> Besides the above command options, a pinning feature is introduced starting from 2.7.0 to prevent certain replicas from getting moved by balancer/mover. This pinning feature is disabled by default, and can be enabled by configuration property “dfs.datanode.block-pinning.enabled”. When enabled, this feature only affects blocks that are written to favored nodes specified in the create() call. This feature is useful when we want to maintain the data locality, for applications such as HBase regionserver.

除了上述命令选项外，从 2.7.0 开始引入了一个固定特性，以防止某些副本被均衡器/移动器移动。

默认情况下该固定功能是禁用的，可以通过配置属性` dfs.datanode.block-pinning.enabled`启用。

当启用时，该特性只影响 create() 调用中指定的优先节点的块。当我们希望为 HBase regionserver 等应用程序维护数据局部性时，该特性非常有用。

### dfsadmin

Usage:
	
    hdfs dfsadmin [-report [-live] [-dead] [-decommissioning] [-enteringmaintenance] [-inmaintenance]]
    ...
    hdfs dfsadmin [-refreshNodes]
    ...
    hdfs dfsadmin [-refreshServiceAcl]

COMMAND_OPTION | Description
---|:---
`-report [-live] [-dead] [-decommissioning] [-enteringmaintenance] [-inmaintenance]`  | Reports basic filesystem information and statistics, The dfs usage can be different from “du” usage, because it measures raw space used by replication, checksums, snapshots and etc. on all the DNs. Optional flags may be used to filter the list of displayed DataNodes.【报告基本的文件系统信息和统计信息，dfs 用法可能与“du”用法不同，因为它度量所有DNs上的副本、校验和、快照等所使用的原始空间。可选标志可用于过滤显示的datanode列表。】
-refreshNodes | Re-read the hosts and exclude files to update the set of Datanodes that are allowed to connect to the Namenode and those that should be decommissioned or recommissioned.【重新读取主机并排除文件，以更新允许连接到Namenode的Datanodes，以及应该退役或重新启用的Datanodes。】
-refreshServiceAcl  |  Reload the service-level authorization policy file.

### diskbalancer

Usage:

   hdfs diskbalancer
     [-plan <datanode> -fs <namenodeURI>]
     [-execute <planfile>]
     [-query <datanode>]
     [-cancel <planfile>]
     [-cancel <planID> -node <datanode>]
     [-report -node <file://> | [<DataNodeID|IP|Hostname>,...]]
     [-report -node -top <topnum>]

COMMAND_OPTION | Description
---|:---
-plan     | Creates a disbalancer plan
-execute  |  Executes a given plan on a datanode
-query    |  Gets the current diskbalancer status from a datanode
-cancel   | Cancels a running plan
-report   | Reports the volume information from datanode(s)

> Runs the diskbalancer CLI. See [HDFS Diskbalancer](https://hadoop.apache.org/docs/r3.2.1/hadoop-project-dist/hadoop-hdfs/HDFSDiskbalancer.html) for more information on this command.

运行 diskbalancer CLI

### haadmin

Usage:

    hdfs haadmin -transitionToActive <serviceId> [--forceactive]
    hdfs haadmin -transitionToStandby <serviceId>
    hdfs haadmin -transitionToObserver <serviceId>
    hdfs haadmin -failover [--forcefence] [--forceactive] <serviceId> <serviceId>
    hdfs haadmin -getServiceState <serviceId>
    hdfs haadmin -getAllServiceState
    hdfs haadmin -checkHealth <serviceId>
    hdfs haadmin -help <command>

COMMAND_OPTION        | Description
---|:---
-checkHealth          |  check the health of the given NameNode【检查指定NameNode的健康状态】
-failover             | initiate a failover between two NameNodes【启动两个namenode间的故障切换】
-getServiceState      |  determine whether the given NameNode is Active or Standby【确定给定的NameNode是Active还是Standby】
-getAllServiceState   |  returns the state of all the NameNodes【返回所有namenode的状态】
-transitionToActive   |  transition the state of the given NameNode to Active (Warning: No fencing is done)【将指定的NameNode状态转换为Active(警告:没有fencing is done)】
-transitionToStandby  |   transition the state of the given NameNode to Standby (Warning: No fencing is done)【将指定的NameNode状态转换到Standby(警告:没有fencing is done)】
-transitionToObserver |  transition the state of the given NameNode to Observer (Warning: No fencing is done)【将给定的NameNode状态转换到观察者(警告:没有fencing is done)】
-help [cmd]           | Displays help for the given command or all commands if none is specified.【显示指定命令或所有命令的帮助。】

See [HDFS HA with NFS](https://hadoop.apache.org/docs/r3.2.1/hadoop-project-dist/hadoop-hdfs/HDFSHighAvailabilityWithNFS.html#Administrative_commands) or [HDFS HA with QJM](https://hadoop.apache.org/docs/r3.2.1/hadoop-project-dist/hadoop-hdfs/HDFSHighAvailabilityWithQJM.html#Administrative_commands) for more information on this command.

### journalnode

Usage: 

    hdfs journalnode

启动一个journalnode。

> This comamnd starts a journalnode for use with [HDFS HA with QJM](https://hadoop.apache.org/docs/r3.2.1/hadoop-project-dist/hadoop-hdfs/HDFSHighAvailabilityWithQJM.html#Administrative_commands).

### namenode

Usage:

    hdfs namenode [-backup] |
          [-checkpoint] |
          [-format [-clusterid cid ] [-force] [-nonInteractive] ] |
          [-upgrade [-clusterid cid] [-renameReserved<k-v pairs>] ] |
          [-upgradeOnly [-clusterid cid] [-renameReserved<k-v pairs>] ] |
          [-rollback] |
          [-rollingUpgrade <rollback |started> ] |
          [-importCheckpoint] |
          [-initializeSharedEdits] |
          [-bootstrapStandby [-force] [-nonInteractive] [-skipSharedEditsCheck] ] |
          [-recover [-force] ] |
          [-metadataVersion ]

COMMAND_OPTION        | Description
---|:---
-backup  |  Start backup node. 【启动备份节点】
-checkpoint | Start checkpoint node. 【启动checkpoint节点】
-format [-clusterid cid]  |  Formats the specified NameNode. It starts the NameNode, formats it and then shut it down. Will throw NameNodeFormatException if name dir already exist and if reformat is disabled for cluster.【格式化指定的NameNode。它启动NameNode，格式化它，然后关闭它。如果name dir已存在，并且集群的重新格式化被禁用，将抛出NameNodeFormatException，】
`-upgrade [-clusterid cid] [-renameReserved <k-v pairs>]` | Namenode should be started with upgrade option after the distribution of new Hadoop version.【在新的Hadoop版本发布之后，应该带有upgrade选项来启动Namenode】
`-upgradeOnly [-clusterid cid] [-renameReserved <k-v pairs>]` | Upgrade the specified NameNode and then shutdown it.【更新指定的NameNode，然后关闭它。】
-rollback | Rollback the NameNode to the previous version. This should be used after stopping the cluster and distributing the old Hadoop version.【回滚NameNode到先前的版本。这应该在停止集群、分发旧的Hadoop版本之后使用。】
`-rollingUpgrade <rollback|started>` |   See [Rolling Upgrade document](https://hadoop.apache.org/docs/r3.2.1/hadoop-project-dist/hadoop-hdfs/HdfsRollingUpgrade.html#NameNode_Startup_Options) for the detail.
-importCheckpoint | Loads image from a checkpoint directory and save it into the current one. Checkpoint dir is read from property dfs.namenode.checkpoint.dir【从checkpoint目录载入映像，并将其存储到当前的节点。checkpoint目录从`dfs.namenode.checkpoint.dir`指定的目录下读取】
-initializeSharedEdits  | Format a new shared edits dir and copy in enough edit log segments so that the standby NameNode can start up.【格式化一个新的共享编辑目录，并复制足够的编辑日志片段，以便可以启动备用NameNode。】
`-bootstrapStandby [-force] [-nonInteractive] [-skipSharedEditsCheck]` |  Allows the standby NameNode’s storage directories to be bootstrapped by copying the latest namespace snapshot from the active NameNode. This is used when first configuring an HA cluster. The option `-force` or `-nonInteractive` has the same meaning as that described in namenode `-format` command. `-skipSharedEditsCheck` option skips edits check which ensures that we have enough edits already in the shared directory to start up from the last checkpoint on the active.【允许备用NameNode的存储目录可以通过从活跃NameNode复制最新的命名空间快照来引导。这在首次配置HA集群时使用。选项`-force`或`-nonInteractive`与namenode`-format`命令中描述的含义相同。`-skipSharedEditsCheck`选项跳过edits检查，这确保我们在共享目录中已经有足够的edits，可以从活跃NameNode的最后一个检查点启动。】
-recover [-force] | Recover lost metadata on a corrupt filesystem. See [HDFS User Guide](https://hadoop.apache.org/docs/r3.2.1/hadoop-project-dist/hadoop-hdfs/HdfsUserGuide.html#Recovery_Mode) for the detail.【恢复在损坏的文件系统上丢失的元数据。】
-metadataVersion  | Verify that configured directories exist, then print the metadata versions of the software and the image.【验证已配置的目录是否存在，然后打印软件和映像的元数据版本。】

> Runs the namenode. More info about the upgrade and rollback is at [Upgrade Rollback](https://hadoop.apache.org/docs/r3.2.1/hadoop-project-dist/hadoop-hdfs/HdfsUserGuide.html#Upgrade_and_Rollback).

运行 namenode。升级和回退的更多信息在 Upgrade Rollback。

## 4、Debug Commands