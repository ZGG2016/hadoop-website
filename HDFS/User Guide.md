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

- Checkpoint node：执行命名空间的周期性检查点，帮助最小化存储在 NameNode 上的日志大小，其中日志包含了对 HDFS 的更改。取代了以前由 Secondary NameNode 所扮演的角色，尽管还没有经过战斗的磨练。NameNode 允许同时有多个 Checkpoint nodes，只要没有向系统注册 Backup nodes。

> Backup node: An extension to the Checkpoint node. In addition to checkpointing it also receives a stream of edits from the NameNode and maintains its own in-memory copy of the namespace, which is always in sync with the active NameNode namespace state. Only one Backup node may be registered with the NameNode at once.

- Backup node：对 Checkpoint node 的扩展。除了进行 checkpoint 之外，它还接收来自 NameNode 的 edits 流，并维护自己的名称空间的内存副本，该副本始终与活跃的 NameNode 名称空间状态同步。NameNode 一次只能注册一个 Backup node。

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

## 7、Checkpoint Node

## 8、Backup Node

## 9、Import Checkpoint

## 10、Balancer

## 11、Rack Awareness

## 12、Safemode

## 13、fsck

## 14、fetchdt

## 15、Recovery Mode

## 16、Upgrade and Rollback

## 17、DataNode Hot Swap Drive

## 18、File Permissions and Security

## 19、Scalability

## 20、Related Documentation