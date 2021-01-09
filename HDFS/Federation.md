# HDFS Federation

[TOC]

> This guide provides an overview of the HDFS Federation feature and how to configure and manage the federated cluster.

本文档提供了 HDFS Federation 特征的概述和如何配置、管理联邦集群。

## 1、Background

![federation-background](./image/federation-background.png)

> HDFS has two main layers:

HDFS 有两个主要的层次：

- 名称空间

	- 由目录、文件和块组成。

	- 支持所有与名称空间相关的文件系统操作，如创建、删除、修改、列出文件和目录。

- 块存储服务，包括两个部分：

	- 块管理（在 Namenode 中执行）

		- 通过处理注册和定期心跳，来提供 Datanode 集群成员资格。
		- 处理块报告并维护块的位置。
		- 支持块的创建、删除、修改、获取块位置等操作。
		- 管理副本的放置位置、复制未达到副本因子的块以及删除过度复制的块。

	- 存储：由 Datanodes 提供，存储块在本地文件系统上，并允许读写访问。

> Namespace

> Consists of directories, files and blocks.

> It supports all the namespace related file system operations such as create, delete, modify and list files and directories.

> Block Storage Service, which has two parts:

> Block Management (performed in the Namenode)

> Provides Datanode cluster membership by handling registrations, and periodic heart beats.

> Processes block reports and maintains location of blocks.

> Supports block related operations such as create, delete, modify and get block location.

> Manages replica placement, block replication for under replicated blocks, and deletes blocks that are over replicated.

> Storage - is provided by Datanodes by storing blocks on the local file system and allowing read/write access.

> The prior HDFS architecture allows only a single namespace for the entire cluster. In that configuration, a single Namenode manages the namespace. HDFS Federation addresses this limitation by adding support for multiple Namenodes/namespaces to HDFS.

以前的 HDFS 架构只允许整个集群使用一个命名空间。在那个配置中，一个 Namenode 管理名称空间。

HDFS Federation 通过向 HDFS 添加对多个 Namenodes/namespaces 的支持来解决这个限制。

## 2、Multiple Namenodes/Namespaces

> In order to scale the name service horizontally, federation uses multiple independent Namenodes/namespaces. The Namenodes are federated; the Namenodes are independent and do not require coordination with each other. The Datanodes are used as common storage for blocks by all the Namenodes. Each Datanode registers with all the Namenodes in the cluster. Datanodes send periodic heartbeats and block reports. They also handle commands from the Namenodes.

**为了横向扩展名称服务，Federation 使用多个独立的 Namenodes/namespaces**。Namenodes 是联合的；Namenodes 是独立的，**不需要相互协调**。

**Datanodes 被所有 Namenodes 用作块的公共存储**。每个 Datanode 向集群中的所有 Namenodes 注册。 Datanodes 定时发送心跳和块报告。它们还处理来自 Namenodes 的命令。

> Users may use [ViewFs](https://hadoop.apache.org/docs/r3.2.1/hadoop-project-dist/hadoop-hdfs/ViewFs.html) to create personalized namespace views. ViewFs is analogous to client side mount tables in some Unix/Linux systems.

用户可以**使用 ViewFs 创建个性化的名称空间视图**。ViewFs 类似于某些 Unix/Linux 系统中的客户端挂载表。

![federation](./image/federation.png)

**Block Pool**

> A Block Pool is a set of blocks that belong to a single namespace. Datanodes store blocks for all the block pools in the cluster. Each Block Pool is managed independently. This allows a namespace to generate Block IDs for new blocks without the need for coordination with the other namespaces. A Namenode failure does not prevent the Datanode from serving other Namenodes in the cluster.

**一个 Block Pool 是属于一个名称空间的一组块**。Datanodes 存储集群中所有 Block Pool 的块。

每个 Block Pool 是独立管理的。这允许名称空间为新块生成块id，而不需要与其他名称空间协调。

一个 Namenode 故障不影响 Datanode 服务于集群内其他 Namenodes。

> A Namespace and its block pool together are called Namespace Volume. It is a self-contained unit of management. When a Namenode/namespace is deleted, the corresponding block pool at the Datanodes is deleted. Each namespace volume is upgraded as a unit, during cluster upgrade.

**ClusterID**

> A ClusterID identifier is used to identify all the nodes in the cluster. When a Namenode is formatted, this identifier is either provided or auto generated. This ID should be used for formatting the other Namenodes into the cluster.

**ClusterID 标识符用于标识集群中的所有节点**。格式化一个 Namenode 时，要么提供该标识符，要么自动生成该标识符。这个 ID 应该用于将其他 Namenodes 格式化到集群中。

### 2.1、Key Benefits

> Namespace Scalability - Federation adds namespace horizontal scaling. Large deployments or deployments using lot of small files benefit from namespace scaling by allowing more Namenodes to be added to the cluster.

名称空间可伸缩性：Federation 添加了名称空间水平可伸缩性。**通过允许向集群中添加更多的 Namenodes 来扩展名称空间**可以使大型部署或使用大量小文件的部署受益。

> Performance - File system throughput is not limited by a single Namenode. Adding more Namenodes to the cluster scales the file system read/write throughput.

性能：**文件系统吞吐量不受单个 Namenode 的限制**。在集群中增加更多的 Namenodes 可以扩展文件系统的读写吞吐量。

> Isolation - A single Namenode offers no isolation in a multi user environment. For example, an experimental application can overload the Namenode and slow down production critical applications. By using multiple Namenodes, different categories of applications and users can be isolated to different namespaces.

隔离：在多用户环境中，单个 Namenode 无法提供隔离。例如，一个实验性的应用程序可能会使Namenode过载，并降低对生产至关重要的应用程序的速度。**通过使用多个 Namenodes，可以将不同类别的应用程序和用户隔离到不同的名称空间**。

## 3、Federation Configuration

> Federation configuration is backward compatible and allows existing single Namenode configurations to work without any change. The new configuration is designed such that all the nodes in the cluster have the same configuration without the need for deploying different configurations based on the type of the node in the cluster.

Federation 配置是向后兼容的，**允许现有的单个 Namenode 配置不需要任何更改就可以工作**。设计的新配置使集群中的所有节点具有相同的配置，而不需要根据集群中节点的类型，部署不同的配置。

> Federation adds a new NameServiceID abstraction. A Namenode and its corresponding secondary/backup/checkpointer nodes all belong to a NameServiceId. In order to support a single configuration file, the Namenode and secondary/backup/checkpointer configuration parameters are suffixed with the NameServiceID.

Federation **添加了一个新的 NameServiceID 抽象**。一个 Namenode 及其对应的 secondary/backup/checkpointer 节点都属于一个 NameServiceId。为了支持单个配置文件，Namenode 和 secondary/backup/checkpointer 配置参数后缀为 NameServiceID。

### 3.1、Configuration:

> Step 1: Add the dfs.nameservices parameter to your configuration and configure it with a list of comma separated NameServiceIDs. This will be used by the Datanodes to determine the Namenodes in the cluster.

步骤1：将 `dfs.nameservices` 参数添加到配置中，并将其指定为以逗号分隔的 NameServiceIDs 列表。这将被 Datanodes 用来确定集群中的 Namenodes。

> Step 2: For each Namenode and Secondary Namenode/BackupNode/Checkpointer add the following configuration parameters suffixed with the corresponding NameServiceID into the common configuration file:

步骤2：对于每个 Namenode 和 Secondary Namenode/BackupNode/Checkpointer，在公共配置文件中添加以下以 NameServiceID 为后缀的配置参数:

Daemon            |  Configuration Parameter
---               |:---
Namenode          |  dfs.namenode.rpc-address dfs.namenode.servicerpc-address dfs.namenode.http-address dfs.namenode.https-address dfs.namenode.keytab.file dfs.namenode.name.dir dfs.namenode.edits.dir dfs.namenode.checkpoint.dir dfs.namenode.checkpoint.edits.dir
Secondary Namenode|	 dfs.namenode.secondary.http-address dfs.secondary.namenode.keytab.file
BackupNode        |  dfs.namenode.backup.address dfs.secondary.namenode.keytab.file

> Here is an example configuration with two Namenodes:

下面是对两个 Namenodes 的配置：

```xml
<configuration>
  <property>
    <name>dfs.nameservices</name>
    <value>ns1,ns2</value>
  </property>
  <property>
    <name>dfs.namenode.rpc-address.ns1</name>
    <value>nn-host1:rpc-port</value>
  </property>
  <property>
    <name>dfs.namenode.http-address.ns1</name>
    <value>nn-host1:http-port</value>
  </property>
  <property>
    <name>dfs.namenode.secondary.http-address.ns1</name>
    <value>snn-host1:http-port</value>
  </property>
  <property>
    <name>dfs.namenode.rpc-address.ns2</name>
    <value>nn-host2:rpc-port</value>
  </property>
  <property>
    <name>dfs.namenode.http-address.ns2</name>
    <value>nn-host2:http-port</value>
  </property>
  <property>
    <name>dfs.namenode.secondary.http-address.ns2</name>
    <value>snn-host2:http-port</value>
  </property>

  .... Other common configuration ...
</configuration>
```

### 3.2、Formatting Namenodes

> Step 1: Format a Namenode using the following command:

步骤1：使用下面的命令格式化一个 Namenode：

	[hdfs]$ $HADOOP_HOME/bin/hdfs namenode -format [-clusterId <cluster_id>]

> Choose a unique cluster_id which will not conflict other clusters in your environment. If a cluster_id is not provided, then a unique one is auto generated.

选择一个唯一的 cluster_id，它不会和你的环境中的其他集群冲突。如果没有提供 cluster_id，那么会自动生成一个唯一的一个 cluster_id。

> Step 2: Format additional Namenodes using the following command:

步骤2：使用下面的命令格式化另外的 Namenodes：

	[hdfs]$ $HADOOP_HOME/bin/hdfs namenode -format -clusterId <cluster_id>

> Note that the cluster_id in step 2 must be same as that of the cluster_id in step 1. If they are different, the additional Namenodes will not be part of the federated cluster.

主要：步骤 2 中的 cluster_id 必须和步骤 1 中的相同。如果不同，另外的 Namenodes 就不会成为联合集群的一部分。

### 3.3、Upgrading from an older release and configuring federation

> Older releases only support a single Namenode. Upgrade the cluster to newer release in order to enable federation During upgrade you can provide a ClusterID as follows:

更旧的版本仅支持一个 Namenode。为了在更新中启用 federation，更新集群到新的版本，你可以提供如下的 ClusterID:

	[hdfs]$ $HADOOP_HOME/bin/hdfs --daemon start namenode -upgrade -clusterId <cluster_ID>

> If cluster_id is not provided, it is auto generated.

如果没有提供 cluster_id，那么会自动生成。

### 3.4、Adding a new Namenode to an existing HDFS cluster

> Perform the following steps:

请执行以下步骤:

> Add dfs.nameservices to the configuration.

- 将 `dfs.namesservices` 添加到配置中。

> Update the configuration with the NameServiceID suffix. Configuration key names changed post release 0.20. You must use the new configuration parameter names in order to use federation.

- 使用 NameServiceID 后缀更新配置。配置密钥的名称在 0.20 版本后发生了变化。必须使用新的配置参数名才能使用 federation。

> Add the new Namenode related config to the configuration file.

- 在配置文件中添加新的与 Namenode 相关的配置。

> Propagate the configuration file to the all the nodes in the cluster.

- 将配置文件传播到集群中的所有节点。

> Start the new Namenode and Secondary/Backup.

- 启动新的 Namenode 和 Secondary/Backup。

> Refresh the Datanodes to pickup the newly added Namenode by running the following command against all the Datanodes in the cluster:

- 对集群中所有的 Datanodes 执行以下命令，刷新 Datanodes 以获取新增的 Namenode。

		[hdfs]$ $HADOOP_HOME/bin/hdfs dfsadmin -refreshNamenodes <datanode_host_name>:<datanode_rpc_port>

## 4、Managing the cluster

### 4.1、Starting and stopping cluster

> To start the cluster run the following command:

运行如下命令启动集群。

	[hdfs]$ $HADOOP_HOME/sbin/start-dfs.sh

> To stop the cluster run the following command:

运行如下命令停止集群。

	[hdfs]$ $HADOOP_HOME/sbin/stop-dfs.sh

> These commands can be run from any node where the HDFS configuration is available. The command uses the configuration to determine the Namenodes in the cluster and then starts the Namenode process on those nodes. The Datanodes are started on the nodes specified in the workers file. The script can be used as a reference for building your own scripts to start and stop the cluster.

这些命令可以在任何有 HDFS 配置的节点上运行。该命令通过配置确定集群中的 Namenode，然后在这些节点上启动 Namenode 进程。Datanodes 在 workers 文件中指定的节点上启动。该脚本可以作为构建你自己的启动和停止集群的脚本的参考。

### 4.2、Balancer

> The Balancer has been changed to work with multiple Namenodes. The Balancer can be run using the command:

为了与多个 Namenodes 一起工作，Balancer 已更改。Balancer 可以使用以下命令运行:

	[hdfs]$ $HADOOP_HOME/bin/hdfs --daemon start balancer [-policy <policy>]

> The policy parameter can be any of the following:

policy 参数可以是以下任意一种:

- datanode：这是默认策略。这将平衡 Datanode 级别的存储。这类似于以前版本中的平衡策略。

- blockpool：在 block pool 级平衡存储，同时在 Datanode 级平衡存储

> datanode - this is the default policy. This balances the storage at the Datanode level. This is similar to balancing policy from prior releases.

> blockpool - this balances the storage at the block pool level which also balances at the Datanode level.

> Note that Balancer only balances the data and does not balance the namespace. For the complete command usage, see [balancer](https://hadoop.apache.org/docs/r3.2.1/hadoop-project-dist/hadoop-hdfs/HDFSCommands.html#balancer).

注意：Balancer 只平衡数据，而不平衡名称空间。有关完整的命令用法，请参见 balancer。

### 4.3、Decommissioning

> Decommissioning is similar to prior releases. The nodes that need to be decomissioned are added to the exclude file at all of the Namenodes. Each Namenode decommissions its Block Pool. When all the Namenodes finish decommissioning a Datanode, the Datanode is considered decommissioned.

退役与以前的版本类似。**需要退役的节点被添加到所有 Namenodes 的排除文件中**。每个 Namenode 都会解除它的 Block Pool。当所有的 Namenodes 都完成了一个 Datanode 的退役，则认为该 Datanode 已经退役。

> Step 1: To distribute an exclude file to all the Namenodes, use the following command:

步骤1：将排除文件分发给所有 Namenodes，使用如下命令:

	[hdfs]$ $HADOOP_HOME/sbin/distribute-exclude.sh <exclude_file>

> Step 2: Refresh all the Namenodes to pick up the new exclude file:

步骤2：刷新所有 Namenodes 以获取新的排除文件:

	[hdfs]$ $HADOOP_HOME/sbin/refresh-namenodes.sh

> The above command uses HDFS configuration to determine the configured Namenodes in the cluster and refreshes them to pick up the new exclude file.

上面的命令使用 HDFS 配置来确定集群中配置的 Namenodes，并刷新它们以获取新的排除文件。

### 4.4、Cluster Web Console

> Similar to the Namenode status web page, when using federation a Cluster Web Console is available to monitor the federated cluster at http://<any_nn_host:port>/dfsclusterhealth.jsp. Any Namenode in the cluster can be used to access this web page.

与 Namenode 状态 web 页面类似，**在使用联合集群时，可以在 `http://<any_nn_host:port>/dfsclusterhealth.jsp` 上使用集群 web 控制台监视联合集群**。集群中的任何 Namenode 都可以用来访问这个 web 页面。

> The Cluster Web Console provides the following information:

集群 Web 控制台提供以下信息:

> A cluster summary that shows the number of files, number of blocks, total configured storage capacity, and the available and used storage for the entire cluster.

- 显示整个集群的文件数、块数、总配置存储容量、可用存储和已使用存储的集群概要。

> A list of Namenodes and a summary that includes the number of files, blocks, missing blocks, and live and dead data nodes for each Namenode. It also provides a link to access each Namenode’s web UI.

- 一个 Namenode 列表和一个摘要，其中包括每个 Namenode 的文件、块、丢失块的数量以及活数据节点和死数据节点的数量。它还提供了一个链接来访问每个 Namenode 的 web UI。

> The decommissioning status of Datanodes.

- Datanodes 的退出状态。