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


### dfs

Usage: 

    hdfs dfs [COMMAND [COMMAND_OPTIONS]]

在 hadoop 支持的文件系统上运行一个文件系统命令。

> Run a filesystem command on the file system supported in Hadoop. The various COMMAND_OPTIONS can be found at [File System Shell Guide](https://hadoop.apache.org/docs/r3.2.1/hadoop-project-dist/hadoop-common/FileSystemShell.html).

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
	
	...
	hdfs dfsadmin [-refreshNodes]
	...
	hdfs dfsadmin [-refreshServiceAcl]

COMMAND_OPTION | Description
---|:---
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

## 4、Debug Commands