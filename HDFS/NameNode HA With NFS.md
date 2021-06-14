# NameNode HA With NFS

[TOC]

## 1、Purpose

> This guide provides an overview of the HDFS High Availability (HA) feature and how to configure and manage an HA HDFS cluster, using NFS for the shared storage required by the NameNodes.

本指南提供了 HDFS 高可用性特征的预览，以及如何为 NameNodes 要求的共享存储而使用 NFS
配置、管理高可用的 HDFS 集群。

> This document assumes that the reader has a general understanding of general components and node types in an HDFS cluster. Please refer to the HDFS Architecture guide for details.

## 2、Note: Using the Quorum Journal Manager or Conventional Shared Storage

> This guide discusses how to configure and use HDFS HA using a shared NFS directory to share edit logs between the Active and Standby NameNodes. For information on how to configure HDFS HA using the Quorum Journal Manager instead of NFS, please see [this alternative guide](https://hadoop.apache.org/docs/r3.2.1/hadoop-project-dist/hadoop-hdfs/HDFSHighAvailabilityWithQJM.html).


本指南讨论如何使用一个共享的 NFS 目录来配置使用 HDFS 的高可用，以在Active NameNode 和
Standby NameNode 间共享 edit logs 。使用 Quorum Journal Manager 实现高可用请见另一份指南。

## 3、Background

> Prior to Hadoop 2.0.0, the NameNode was a single point of failure (SPOF) in an HDFS cluster. Each cluster had a single NameNode, and if that machine or process became unavailable, the cluster as a whole would be unavailable until the NameNode was either restarted or brought up on a separate machine.

在 Hadoop 2.0.0 之前，在 HDFS 集群中，NameNode 有单点故障。每个集群有一个 NameNode，如果那台机器或进程不可用，整个集群就会不可用，直到重启了 NameNode，或在另一台机器上启动。

> This impacted the total availability of the HDFS cluster in two major ways:

这通过以下两种方式来影响 HDFS 集群的可用性：

- 在计划外的事件中，如机器宕机，集群将不可用，直到重启 NameNode。

- 计划中的维护事件，如在 NameNode 机器上的软件或硬件升级，将导致集群窗口停机。

> In the case of an unplanned event such as a machine crash, the cluster would be unavailable until an operator restarted the NameNode.

> Planned maintenance events such as software or hardware upgrades on the NameNode machine would result in windows of cluster downtime.

> The HDFS High Availability feature addresses the above problems by providing the option of running two (or more, as of Hadoop 3.0.0) redundant NameNodes in the same cluster in an Active/Passive configuration with a hot standby(s). This allows a fast failover to a new NameNode in the case that a machine crashes, or a graceful administrator-initiated failover for the purpose of planned maintenance.

HDFS 高可用性特征通过在相同集群下允许两个（从 3.0.0 开始，或者更多）冗余的 NameNodes 来解决上述问题，一个 Active ，一个 Passive，并保持热备份。

这允许在机器宕机时，往新的 NameNode 上进行一次快速故障转移，或者，在计划内的维护时，进行由管理员发起的故障转移。

## 4、Architecture

> In a typical HA cluster, two or more separate machines are configured as NameNodes. At any point in time, exactly one of the NameNodes is in an Active state, and the others are in a Standby state. The Active NameNode is responsible for all client operations in the cluster, while the Standby is simply acting as a slave, maintaining enough state to provide a fast failover if necessary.

在一个 HA 集群中，配置两个或更多独立的机器作为 NameNode 节点。**在任意时刻，一个处在 Active 状态，其他的处在 Standyby 状态**。
Active NameNode 负责处理客户端操作，Standyby NameNode 仅仅作为一个工作节点维持自身足够的状态，
在必要时候，实现快速故障转移。

> In order for the Standby nodes to keep their state synchronized with the Active node, the current implementation requires that the nodes have access to a directory on a shared storage device (eg an NFS mount from a NAS). This restriction will likely be relaxed in future versions.

**Standby 节点为了和 Active 节点保持状态同步，当前实现要求节点可以访问共享存储设备上的一个目录**（如，从 NAS 的 NFS 挂载）。在未来版本会放松这个限制。 

> When any namespace modification is performed by the Active node, it durably logs a record of the modification to an edit log file stored in the shared directory. The Standby nodes are constantly watching this directory for edits, and as it sees the edits, it applies them to its own namespace. In the event of a failover, the Standby will ensure that it has read all of the edits from the shared storage before promoting itself to the Active state. This ensures that the namespace state is fully synchronized before a failover occurs.

**当在 Active 节点修改任意的名称空间时，它都会将修改记录记录到存储在共享目录的 edit log 文件中**。

Standby 节点持续观察啊这个目录，一旦发现新的编辑记录，它就将其应用到自己的名称空间。

在故障转移事件中，Standby 节点在成为 Active 状态前，会确保读取了共享存储中的所有编辑记录。这就确保了名称空间状态在故障转移之前保持完全同步。

> In order to provide a fast failover, it is also necessary that the Standby nodes have up-to-date information regarding the location of blocks in the cluster. In order to achieve this, the DataNodes are configured with the location of all NameNodes, and send block location information and heartbeats to all the NameNodes.

为了实现快速故障转移，Standy 节点需要知道块位置的最新信息。

为了实现这个，**DataNode 会有所有
NameNode 的位置，并发送块的位置信息和心跳给它们**。

It is vital for the correct operation of an HA cluster that only one of the NameNodes be Active at a time. Otherwise, the namespace state would quickly diverge between the two, risking data loss or other incorrect results. In order to ensure this property and prevent the so-called “split-brain scenario,” the administrator must configure at least one fencing method for the shared storage. During a failover, if it cannot be verified that the previous Active node has relinquished its Active state, the fencing process is responsible for cutting off the previous Active’s access to the shared edits storage. This prevents it from making any further edits to the namespace, allowing the new Active to safely proceed with failover.

保持集群在一时间点仅有一个 Active NameNode 是非常重要的。否则名称空间被划分成了两部分，会数据丢失或出现其他错误结果的风险。

**为了确保这个性质并阻止 "split_brain scenario"，管理员必须为共享存储配置至少一个 fencing method**。

在故障转移期间，如果它不能验证前面的 Active 节点已不处在活跃状态，fencing 进程就负责切断前面的 Active 节点对共享编辑存储的访问。这就有效阻止了
其他 NameNode 继续保持 Active 状态，允许新的 Active NameNode 安全的处理故障转移。

## 5、Hardware resources

> In order to deploy an HA cluster, you should prepare the following:

为了部署高可用集群，你应该做如下啊准备：

> NameNode machines - the machines on which you run the Active and Standby NameNodes should have equivalent hardware to each other, and equivalent hardware to what would be used in a non-HA cluster.

- NameNode 机器：运行 Active 和 Standby NameNodes 的机器，相同的硬件配置。

> Shared storage - you will need to have a shared directory which the NameNode machines have read/write access to. Typically this is a remote filer which supports NFS and is mounted on each of the NameNode machines. Currently only a single shared edits directory is supported. Thus, the availability of the system is limited by the availability of this shared edits directory, and therefore in order to remove all single points of failure there needs to be redundancy for the shared edits directory. Specifically, multiple network paths to the storage, and redundancy in the storage itself (disk, network, and power). Because of this, it is recommended that the shared storage server be a high-quality dedicated NAS appliance rather than a simple Linux server.

- 共享存储：需要一个共享目录， NameNode 机器对其有读写权限。

	这是一个远程文件，支持 NFS，挂载在其中一个 NameNode 机器上。

	当前，仅支持一个共享编辑目录。因此，系统的可用性被这个共享编辑目录的可用性所限制。

	为了移除单点故障，需要有冗余的共享目录。具体来说就是，到存储设备的多个网络路径，以及存储本身的冗余(磁盘、网络和电源)。

	因此，**建议共享存储服务器是高质量的专用 NAS 设备，而不是简单的 Linux 服务器**。

> Note that, in an HA cluster, the Standby NameNodes also perform checkpoints of the namespace state, and thus it is not necessary to run a Secondary NameNode, CheckpointNode, or BackupNode in an HA cluster. In fact, to do so would be an error. This also allows one who is reconfiguring a non-HA-enabled HDFS cluster to be HA-enabled to reuse the hardware which they had previously dedicated to the Secondary NameNode.

**在高可用集群中，Standby NameNodes 也执行名称空间状态的 checkpoint 过程。因此在高可用集群中不需要运行 SecondryNameNode、CheckpointNode、BackupNode**。

事实上，如果设置了这些节点，会出现错误。

这也允许将非高可用集群重新配置成高可用集群，这样就可以重用 Secondary NameNode 所在的机器资源。

## 6、Deployment

### 6.1、Configuration overview

> Similar to Federation configuration, HA configuration is backward compatible and allows existing single NameNode configurations to work without change. The new configuration is designed such that all the nodes in the cluster may have the same configuration without the need for deploying different configuration files to different machines based on the type of the node.

和 Federation 配置类似，HA 配置向后兼容，可以使已存在的单个 NameNode 配置不需修改就可以工作。一个新的配置特点就是集群中的所有节点都有相同的配置。

> Like HDFS Federation, HA clusters reuse the nameservice ID to identify a single HDFS instance that may in fact consist of multiple HA NameNodes. In addition, a new abstraction called NameNode ID is added with HA. Each distinct NameNode in the cluster has a different NameNode ID to distinguish it. To support a single configuration file for all of the NameNodes, the relevant configuration parameters are suffixed with the nameservice ID as well as the NameNode ID.

和 HDFS Federation 一样，HA 集群也**使用 nameservice ID 识别一个 HDFS 实例** ，一个 HDFS 实例可能由多个 HA NameNodes 组成。

HA 集群中，新增加了一个 NameNode ID 的概念。**集群中的每个 NameNode 都有一个独立的 NameNode ID**。

为了支持所有的 NameNode 使用相同的配置文件，相关的配置参数都以 nameservice ID 和 NameNode ID 做后缀。

### 6.2、Configuration details

> To configure HA NameNodes, you must add several configuration options to your hdfs-site.xml configuration file.

配置 hdfs-site.xml 文件。

> The order in which you set these configurations is unimportant, but the values you choose for dfs.nameservices and dfs.ha.namenodes.[nameservice ID] will determine the keys of those that follow. Thus, you should decide on these values before setting the rest of the configuration options.

配置属性的顺序不重要。但是要先配置的 `dfs.nameservices` 和 `dfs.ha.namenodes.[nameservice ID]`。 

- dfs.nameservices - the logical name for this new nameservice

> Choose a logical name for this nameservice, for example “mycluster”, and use this logical name for the value of this config option. The name you choose is arbitrary. It will be used both for configuration and as the authority component of absolute HDFS paths in the cluster.

为这个 nameservice 选择一个逻辑名称，例如"mycluster"，这个名称用在配置文件中，或作为 HDFS 路径的一部分。

> Note: If you are also using HDFS Federation, this configuration setting should also include the list of other nameservices, HA or otherwise, as a comma-separated list.

注意：如果也在使用 HDFS federation，这个配置的设置应该包含其他 nameservices，用逗号分隔。

```xml
<property>
  <name>dfs.nameservices</name>
  <value>mycluster</value>
</property>
```

- dfs.ha.namenodes.[nameservice ID] - unique identifiers for each NameNode in the nameservice

> Configure with a list of comma-separated NameNode IDs. This will be used by DataNodes to determine all the NameNodes in the cluster. For example, if you used “mycluster” as the nameservice ID previously, and you wanted to use “nn1”,“nn2” and “nn3” as the individual IDs of the NameNodes, you would configure this as such:

配置为逗号分隔的 NameNode IDs 列表。

让 DataNodes 知道哪些是 NameNodes。 如果你使用 “mycluster” 作为 nameservice ID，那么每个 nameservice ID 可以设为 “nn1”, “nn2” and “nn3”。

```xml
<property>
  <name>dfs.ha.namenodes.mycluster</name>
  <value>nn1,nn2,nn3</value>
</property>
```

> Note: The minimum number of NameNodes for HA is two, but you can configure more. Its suggested to not exceed 5 - with a recommended 3 NameNodes - due to communication overheads.

注意：高可用模式下，最少使用两个 NameNodes ，但是你可以配置更多。推荐设为 3 个，考虑到通信开销，最好不超过 5.

- dfs.namenode.rpc-address.[nameservice ID].[name node ID] - the fully-qualified RPC address for each NameNode to listen on

> For both of the previously-configured NameNode IDs, set the full address and IPC port of the NameNode process. Note that this results in two separate configuration options. For example:

对前面配置的 NameNode IDs，设置 NameNode 进程的完整的地址和 IPC 端口。

注意，这是两个独立的配置项。

```xml
<property>
  <name>dfs.namenode.rpc-address.mycluster.nn1</name>
  <value>machine1.example.com:8020</value>
</property>
<property>
  <name>dfs.namenode.rpc-address.mycluster.nn2</name>
  <value>machine2.example.com:8020</value>
</property>
<property>
  <name>dfs.namenode.rpc-address.mycluster.nn3</name>
  <value>machine3.example.com:8020</value>
</property>
```

> Note: You may similarly configure the “servicerpc-address” setting if you so desire.

注意：如果你愿意，也可以配置 “servicerpc-address” 设置。

- dfs.namenode.http-address.[nameservice ID].[name node ID] - the fully-qualified HTTP address for each NameNode to listen on

> Similarly to rpc-address above, set the addresses for both NameNodes’ HTTP servers to listen on. For example:

设置监听的 NameNodes HTTP 服务的地址。

```xml
<property>
  <name>dfs.namenode.http-address.mycluster.nn1</name>
  <value>machine1.example.com:9870</value>
</property>
<property>
  <name>dfs.namenode.http-address.mycluster.nn2</name>
  <value>machine2.example.com:9870</value>
</property>
<property>
  <name>dfs.namenode.http-address.mycluster.nn3</name>
  <value>machine3.example.com:9870</value>
</property>
```

> Note: If you have Hadoop’s security features enabled, you should also set the https-address similarly for each NameNode.

注意：如果你启动了 Hadoop 的安全特性，你也应该同样地为每个 NameNode 设置 https-address 的地址。

- dfs.namenode.shared.edits.dir - the location of the shared storage directory

> This is where one configures the path to the remote shared edits directory which the Standby NameNodes use to stay up-to-date with all the file system changes the Active NameNode makes. You should only configure one of these directories. This directory should be mounted r/w on the NameNode machines. The value of this setting should be the absolute path to this directory on the NameNode machines. For example:

配置远程共享编辑目录的路径，即 Standby NameNodes 用来保持和 Active NameNode 同步的目录。

你应该配置其中一个目录。这个目录应该挂载在 NameNode 机器上，具有读写权限。

这个配置的值是一个 NameNode 机器上的绝对路径。

```xml
<property>
  <name>dfs.namenode.shared.edits.dir</name>
  <value>file:///mnt/filer1/dfs/ha-name-dir-shared</value>
</property>
```

- dfs.client.failover.proxy.provider.[nameservice ID] - the Java class that HDFS clients use to contact the Active NameNode

> Configure the name of the Java class which will be used by the DFS Client to determine which NameNode is the current Active, and therefore which NameNode is currently serving client requests. The two implementations which currently ship with Hadoop are the ConfiguredFailoverProxyProvider and the RequestHedgingProxyProvider (which, for the first call, concurrently invokes all namenodes to determine the active one, and on subsequent requests, invokes the active namenode until a fail-over happens), so use one of these unless you are using a custom proxy provider.

配置被 DFS 客户端用来决定哪一个 NameNode 当前是 Active，从而决定哪一个 NameNode 目前服务于客户端请求。

目前只有两个实现，ConfiguredFailoverProxyProvider 和 the RequestHedgingProxyProvider ，(对于第一个调用，并发地调用所有的 namenode 以确定 Active namenode，并在随后的请求中调用 Active namenode，直到发生故障转移)所以除非你用了自定义的类否则就用其中之一。例如：

```xml
<property>
  <name>dfs.client.failover.proxy.provider.mycluster</name>
  <value>org.apache.hadoop.hdfs.server.namenode.ha.ConfiguredFailoverProxyProvider</value>
</property>
```

- dfs.ha.fencing.methods - a list of scripts or Java classes which will be used to fence the Active NameNode during a failover

> It is critical for correctness of the system that only one NameNode be in the Active state at any given time. Thus, during a failover, we first ensure that the Active NameNode is either in the Standby state, or the process has terminated, before transitioning another NameNode to the Active state. In order to do this, you must configure at least one fencing method. These are configured as a carriage-return-separated list, which will be attempted in order until one indicates that fencing has succeeded. There are two methods which ship with Hadoop: shell and sshfence. For information on implementing your own custom fencing method, see the org.apache.hadoop.ha.NodeFencer class.

在任意时刻只有一个 Active NameNode，这对系统的正确性是可取的。因此，在故障转移期间，我们首先确保，在其他 NameNode 成为 Active 状态前，Active NameNode 要么处在 Standby 状态，要么进程被终止。

为了做到这一点，必须配置至少一个 fencing method。这些被配置为一个 carriage-return-separated 列表，这些将按顺序尝试，直到有一个 fencing 已经成功为止。

Hadoop 附带了两种方法：shell 和 sshfence。有关实现自定义 fencing 方法的信息，请参见 `org.apache.hadoop.ha.NodeFencer` 类。

- - sshfence - SSH to the Active NameNode and kill the process

> The sshfence option SSHes to the target node and uses fuser to kill the process listening on the service’s TCP port. In order for this fencing option to work, it must be able to SSH to the target node without providing a passphrase. Thus, one must also configure the dfs.ha.fencing.ssh.private-key-files option, which is a comma-separated list of SSH private key files. For example:

sshfence 项 SSH 到目标节点，然后用 fuser 命令杀掉监听在服务的 TCP 端口上的进程。

为了使 fencing 项工作，它必须能够免密码 SSH 到目标节点。因此，你必须配置 `dfs.ha.fencing.ssh.private-key-files` 选项，这是一个逗号分隔的 SSH 私钥文件的列表。例如：

```xml
    <property>
      <name>dfs.ha.fencing.methods</name>
      <value>sshfence</value>
    </property>

    <property>
      <name>dfs.ha.fencing.ssh.private-key-files</name>
      <value>/home/exampleuser/.ssh/id_rsa</value>
    </property>
```

> Optionally, one may configure a non-standard username or port to perform the SSH. One may also configure a timeout, in milliseconds, for the SSH, after which this fencing method will be considered to have failed. It may be configured like so:

可选择地，你可以配置一个非标准的用户名或者端口号来执行 SSH 。你也可能为此 SSH 配置一个 超时限制，单位毫秒，超过此超时限制 fencing method 将被认为失败。它可能被配置成这样：

```xml
<property>
  <name>dfs.ha.fencing.methods</name>
  <value>sshfence([[username][:port]])</value>
</property>
<property>
  <name>dfs.ha.fencing.ssh.connect-timeout</name>
  <value>30000</value>
</property>
```

- - shell - run an arbitrary shell command to fence the Active NameNode

> The shell fencing method runs an arbitrary shell command. It may be configured like so:

shell fence method 运行一个任意的 shell 命令，它可能配置成这样：

```xml
<property>
  <name>dfs.ha.fencing.methods</name>
  <value>shell(/path/to/my/script.sh arg1 arg2 ...)</value>
</property>
```

> The string between ‘(’ and ‘)’ is passed directly to a bash shell and may not include any closing parentheses.

括号之间的字符串被直接传给 bash shell，可能不包含任何关闭括号。

> The shell command will be run with an environment set up to contain all of the current Hadoop configuration variables, with the `‘_’` character replacing any ‘.’ characters in the configuration keys. The configuration used has already had any namenode-specific configurations promoted to their generic forms – for example dfs_namenode_rpc-address will contain the RPC address of the target node, even though the configuration may specify that variable as dfs.namenode.rpc-address.ns1.nn1.

Shell 命令将运行在一个包含当前所有 Hadoop 配置变量的环境中，在配置的 key 中用 `_` 代替 `.` 。

所用的配置已经将任何一个 namenode 特定的配置改变成通用的形式。例如 `dfs_namenode_rpc-address` 将包含目标节点的 RPC 地址，即使配置指定的变量可能是 `dfs.namenode.rpc-address.ns1.nn1`。

> Additionally, the following variables referring to the target node to be fenced are also available:

此外，下面的这些涉及到被 fence 的目标节点的变量，也是可用的：

A | B
---|:---
$target_host     |   hostname of the node to be fenced
$target_port     |   IPC port of the node to be fenced
$target_address     |   the above two, combined as host:port
$target_nameserviceid     |   the nameservice ID of the NN to be fenced
$target_namenodeid     |   the namenode ID of the NN to be fenced

> These environment variables may also be used as substitutions in the shell command itself. For example:

这些环境变量可能被用来替换 shell 命令中的变量，例如：

```xml
<property>
  <name>dfs.ha.fencing.methods</name>
  <value>shell(/path/to/my/script.sh --nameservice=$target_nameserviceid $target_host:$target_port)</value>
</property>
```

> If the shell command returns an exit code of 0, the fencing is determined to be successful. If it returns any other exit code, the fencing was not successful and the next fencing method in the list will be attempted.

如果 shell 命令结束返回 0，fence 被认为是成功了。如果返回任何其他的结束码，fence 不是成功的，列表中的下个 fence 方法将被尝试。

> Note: This fencing method does not implement any timeout. If timeouts are necessary, they should be implemented in the shell script itself (eg by forking a subshell to kill its parent in some number of seconds).

注意：这个 fencing method 不实现任何的 timeout。如果 timeout 是必要的，它们应该在 shell 脚本中实现（例如通过 fork subshell 在多少秒后杀死他的父 shell）。

- fs.defaultFS - the default path prefix used by the Hadoop FS client when none is given

> Optionally, you may now configure the default path for Hadoop clients to use the new HA-enabled logical URI. If you used “mycluster” as the nameservice ID earlier, this will be the value of the authority portion of all of your HDFS paths. This may be configured like so, in your core-site.xml file:

可选地，你现在可能为 Hadoop 客户端配置了默认使用的路径来用新的启用 HA 的逻辑 URI。

如果你用 “mycluster” 作为 nameservice ID ，这个 id 将是所有你的 HDFS 路径的 authority 部分。这可能被配置成这样，在你的 core-site.xml 文件里：

```xml
<property>
  <name>fs.defaultFS</name>
  <value>hdfs://mycluster</value>
</property>
```

### 6.3、Deployment details

> After all of the necessary configuration options have been set, one must initially synchronize the two HA NameNodes’ on-disk metadata.

在所有必要的配置被设置之后，首先需要同步两个 HA NameNodes 的磁盘元数据。

> If you are setting up a fresh HDFS cluster, you should first run the format command (hdfs namenode -format) on one of NameNodes.

- 如果你正设置一个新的 HDFS 集群，你应该首先在其中一个 NameNodes 上运行格式化命令。

> If you have already formatted the NameNode, or are converting a non-HA-enabled cluster to be HA-enabled, you should now copy over the contents of your NameNode metadata directories to the other, unformatted NameNodes by running the command “hdfs namenode -bootstrapStandby” on the unformatted NameNode. Running this command will also ensure that the shared edits directory (as configured by dfs.namenode.shared.edits.dir) contains sufficient edits transactions to be able to start both NameNodes.

- 如果你已经格式化了 NameNode ，或者是从一个非 HA 的集群转变成一个 HA 集群，你现在应该**复制你的 NameNode 上的元数据目录到另一个 NameNode，没有格式化的 NameNode 通过在此机器上运行 `hdfs namenode -bootstrapStandby` 命令格式化**。 运行此命令也将确保共享编辑目录（就像通过 dfs.namenode.shared.edits.dir 配置的那样）包含足够的 edit 事务来启动两个 NameNodes。

> If you are converting a non-HA NameNode to be HA, you should run the command “hdfs -initializeSharedEdits”, which will initialize the shared edits directory with the edits data from the local NameNode edits directories.

- 如果你**将一个非 HA 的集群转变为一个 HA 的集群，你应该运行 `hdfs -initializeSharedEdits`命令**，这个命令将会用本地 NameNode 的编辑目录中的数据初始化共享编辑目录。

> At this point you may start all of your HA NameNodes as you normally would start a NameNode.

这个时候，你启动两个 HA NameNode，就像你启动一个 NameNode 一样。

> You can visit each of the NameNodes’ web pages separately by browsing to their configured HTTP addresses. You should notice that next to the configured address will be the HA state of the NameNode (either “standby” or “active”.) Whenever an HA NameNode starts, it is initially in the Standby state.

你可以通过浏览他们配置的 HTTP 地址，访问分别每一个 NameNode 的 web 主页。你应该注意到配置的地址的下面将会是 NameNode 的 HA 状态（Standby或Active）。不管一个 HA NameNode 何时启动，它首先会处在 Standby 状态。

### 6.4、Administrative commands

> Now that your HA NameNodes are configured and started, you will have access to some additional commands to administer your HA HDFS cluster. Specifically, you should familiarize yourself with all of the subcommands of the “hdfs haadmin” command. Running this command without any additional arguments will display the following usage information:

既然你的 HA NameNode 被配置和启动了，你将可以使用一些命令来管理你的 HA HDFS 集群。

特别的，你应该熟悉 `hdfs haadmin ` 命令的所有子命令。不加任何参数运行此命令将显示下面的帮助信息：

	Usage: DFSHAAdmin [-ns <nameserviceId>]
	    [-transitionToActive <serviceId>]
	    [-transitionToStandby <serviceId>]
	    [-failover [--forcefence] [--forceactive] <serviceId> <serviceId>]
	    [-getServiceState <serviceId>]
	    [-getAllServiceState]
	    [-checkHealth <serviceId>]
	    [-help <command>]

> This guide describes high-level uses of each of these subcommands. For specific usage information of each subcommand, you should run “hdfs haadmin -help <command>”.

本指南描述这些子命令的高级用法。对于每一个子命令特殊的用法的信息，你应该运行 `hdfs haadmin -help <command>` 来获得。

- transitionToActive and transitionToStandby - transition the state of the given NameNode to Active or Standby

> These subcommands cause a given NameNode to transition to the Active or Standby state, respectively. These commands do not attempt to perform any fencing, and thus should rarely be used. Instead, one should almost always prefer to use the “hdfs haadmin -failover” subcommand.

这两个子命令使给定的 NameNode 对应转到 Active 或者 Standby 状态。这两个命令不会尝试运行任何的 fencing，因此不应该经常使用。你应该更倾向于用 `hdfs haadmin -failover` 命令。

- failover - initiate a failover between two NameNodes

> This subcommand causes a failover from the first provided NameNode to the second. If the first NameNode is in the Standby state, this command simply transitions the second to the Active state without error. If the first NameNode is in the Active state, an attempt will be made to gracefully transition it to the Standby state. If this fails, the fencing methods (as configured by dfs.ha.fencing.methods) will be attempted in order until one succeeds. Only after this process will the second NameNode be transitioned to the Active state. If no fencing method succeeds, the second NameNode will not be transitioned to the Active state, and an error will be returned.

这个子命令从提供的第一个 NameNode 到第二个 NameNode 发起一次故障转移。

如果第一个 NameNode 是 Standby 状态，这个命令将简单的将第二个 NameNode 转成 Active 状态，不会报错。

如果第一个 NameNode 在 Active 状态，将会尝试优雅的将其转变为 Standby 状态。如果这个过程失败，将会依次使用 fence method（就像 `dfs.ha.fencing.methods` 配置的）直到有一个返回 success 。在这个过程之后，第二个 NameNode 将会被转换为 Active 状态。如果没有 fence method 成功，第二个 NameNode 将不会被转变成 Active 状态，将会返回一个错误。

- getServiceState - determine whether the given NameNode is Active or Standby

> Connect to the provided NameNode to determine its current state, printing either “standby” or “active” to STDOUT appropriately. This subcommand might be used by cron jobs or monitoring scripts which need to behave differently based on whether the NameNode is currently Active or Standby.

连接到给定的 NameNode 来判断它目前的状态，打印 Standby 或者 Active 到合适的标准输出。这个子命令可以被用作定时任务或者监控脚本，根据 NameNode 是 Standby 还是 Active 做出不同行为。

- getAllServiceState - returns the state of all the NameNodes

> Connect to the configured NameNodes to determine the current state, print either “standby” or “active” to STDOUT appropriately.

连接到配置的 NameNode 来判断它目前的状态，打印 “standby” 或者 “active” 到合适的标准输出。

- checkHealth - check the health of the given NameNode

> Connect to the provided NameNode to check its health. The NameNode is capable of performing some diagnostics on itself, including checking if internal services are running as expected. This command will return 0 if the NameNode is healthy, non-zero otherwise. One might use this command for monitoring purposes.

连接到给定的 NameNode 检查它的健康状况。

NameNode 可以在对自己进行一些检测，包括检查内部服务是否正常运行。如果 NameNode 是健康的，这个命令将返回 0，不是则返回非 0 值。你可以用这个命令用于检测目的。

> Note: This is not yet implemented, and at present will always return success, unless the given NameNode is completely down.

注意：这还没有实现，现在将总是返回 success，除非给定的 NameNode 完全关闭。

## 7、Automatic Failover

### 7.1、Introduction

> The above sections describe how to configure manual failover. In that mode, the system will not automatically trigger a failover from the active to the standby NameNode, even if the active node has failed. This section describes how to configure and deploy automatic failover.

上边的部分描述了如何配置一个手工故障转移。在那种模式下，系统将不会自动触发一个故障转移，将一个 NameNode 从 Active 转换成 Standby ，即使 Active 节点已经失效。

这个部分描述了如何配置和部署一个自动故障转移。

### 7.2、Components

> Automatic failover adds two new components to an HDFS deployment: a ZooKeeper quorum, and the ZKFailoverController process (abbreviated as ZKFC).

自动故障转移增加了两个新的组件到 HDFS 的部署中：Zookeeper 仲裁和 ZKFailoverController 进程（简称ZKFC）。

> Apache ZooKeeper is a highly available service for maintaining small amounts of coordination data, notifying clients of changes in that data, and monitoring clients for failures. The implementation of automatic HDFS failover relies on ZooKeeper for the following things:

Apache Zookeeper 是一个维护少量数据一致性的高可用的服务，通知客户端数据的变化，同时监控客户端的失效状况。 HDFS 的自动故障转移的实现依赖 Zookeeper：

> Failure detection - each of the NameNode machines in the cluster maintains a persistent session in ZooKeeper. If the machine crashes, the ZooKeeper session will expire, notifying the other NameNode that a failover should be triggered.

- Failure detection：集群中的每一台 NameNode 机器在 Zookeeper 中保持一个永久的 session。如果这台机器宕机，通知其他的NameNode，故障转移应该被触发了。

> Active NameNode election - ZooKeeper provides a simple mechanism to exclusively elect a node as active. If the current active NameNode crashes, another node may take a special exclusive lock in ZooKeeper indicating that it should become the next active.

- Active NameNode 选举：Zookeeper 提供了一个简单的机制专门用来选择一个节点为 Active。如果目前 Active NameNode 宕机，另一个节点可以获取 Zookeeper 中的一个特殊的排它锁，声明它应该变成下一个 Active NameNode。

> The ZKFailoverController (ZKFC) is a new component which is a ZooKeeper client which also monitors and manages the state of the NameNode. Each of the machines which runs a NameNode also runs a ZKFC, and that ZKFC is responsible for:

ZKFC 是一个新的组件，它是一个 Zookeeper 客户端，同时也用来监视和管理 NameNode 的状态。每一台运行 NameNode 的机器都要同时运行一个 ZKFC 进程，此进程主要负责：

> Health monitoring - the ZKFC pings its local NameNode on a periodic basis with a health-check command. So long as the NameNode responds in a timely fashion with a healthy status, the ZKFC considers the node healthy. If the node has crashed, frozen, or otherwise entered an unhealthy state, the health monitor will mark it as unhealthy.

- 健康监控：ZKFC 周期性地用健康监测命令 ping 它的本地 NameNode 。只要 NameNode 及时地用健康状态响应，ZKFC 就认为此节点是健康的。如果此节点宕机、睡眠或者其他的原因进入了一个不健康的状态，健康监控器将会标记其为不健康。

> ZooKeeper session management - when the local NameNode is healthy, the ZKFC holds a session open in ZooKeeper. If the local NameNode is active, it also holds a special “lock” znode. This lock uses ZooKeeper’s support for “ephemeral” nodes; if the session expires, the lock node will be automatically deleted.

- Zookeeper Session 管理：当本地的 NameNode 是健康的，ZKFC 在 Zookeeper 中保持一个打开的 session。如果本地 NameNode 是 Active 状态，它同时保持一个特殊的 “lock” znode 。这个锁是利用 Zookeeper 对 ephemeral node 的支持；如果 session 过期，这个 lock node 将会自动删除。

> ZooKeeper-based election - if the local NameNode is healthy, and the ZKFC sees that no other node currently holds the lock znode, it will itself try to acquire the lock. If it succeeds, then it has “won the election”, and is responsible for running a failover to make its local NameNode active. The failover process is similar to the manual failover described above: first, the previous active is fenced if necessary, and then the local NameNode transitions to active state.

- 基于 Zookeeper 的选举：如果本地 NameNode 是健康的，ZKFC 看到当前没有其他的节点保持 lock znode，它将自己尝试获取这个锁。如果获取成功，它就赢得了选举，然后负责运行一个故障转移来使它本地的 NameNode 变为 Active。故障转移进程与上边描述的的手工转移的类似：首先，如果有必要，原来 Active NameNode 被 fence ，然后，本地 NameNode 转变为 Active 状态。

> For more details on the design of automatic failover, refer to the design document attached to HDFS-2185 on the Apache HDFS JIRA.

一个参考自动故障转移的设计文档来获取更多的信息，参考Apache HDFS JIRA上的 HDFS-2185 设计文档。

### 7.3、Deploying ZooKeeper

> In a typical deployment, ZooKeeper daemons are configured to run on three or five nodes. Since ZooKeeper itself has light resource requirements, it is acceptable to collocate the ZooKeeper nodes on the same hardware as the HDFS NameNode and Standby Node. Many operators choose to deploy the third ZooKeeper process on the same node as the YARN ResourceManager. It is advisable to configure the ZooKeeper nodes to store their data on separate disk drives from the HDFS metadata for best performance and isolation.

在一个典型的部署中，Zookeeper 守护进程配置为在 3 或 5 个节点上运行。因为 Zookeeper 本身有轻量级的资源需求，将 Zookeeper 守护进程跟 HDFS 的 Active NameNode 和 Standby NameNode 在同一个硬件上是可以接受的。许多管理员也选择将第三个 Zookeeper 进程部署到 YARN ResourceManager 一个节点上。建议将 Zookeeper 节点它们的数据存储到一个单独的磁盘驱动，以为了更好的性能和解耦，与存放 HDFS 元数据的驱动程序分开。

> The setup of ZooKeeper is out of scope for this document. We will assume that you have set up a ZooKeeper cluster running on three or more nodes, and have verified its correct operation by connecting using the ZK CLI.

Zookeeper 的建立超出了本文档的范围。我们将假设你已经建立起了一个 3 个或者更多节点的 ZooKeeper 集群，已经通过用 ZK CLI 连接到 ZKServer 验证了其正确性。

### 7.4、Before you begin

> Before you begin configuring automatic failover, you should shut down your cluster. It is not currently possible to transition from a manual failover setup to an automatic failover setup while the cluster is running.

在你开始配置自动故障转移之前，你应该关闭你的集群。目前，在集群运行过程中，从手工故障转移到自动故障转移的转变是不可能的。

### 7.5、Configuring automatic failover

> The configuration of automatic failover requires the addition of two new parameters to your configuration. In your hdfs-site.xml file, add:

自动故障转移的配置需要增加两个新的参数到你的配置文件中。在hdfs-site.xml中增加：

```xml
 <property>
   <name>dfs.ha.automatic-failover.enabled</name>
   <value>true</value>
 </property>
```

> This specifies that the cluster should be set up for automatic failover. In your core-site.xml file, add:

这指出集群应该被建立为自动故障转移模式。在 core-site.xml 中，增加：

```xml
 <property>
   <name>ha.zookeeper.quorum</name>
   <value>zk1.example.com:2181,zk2.example.com:2181,zk3.example.com:2181</value>
 </property>
```

> This lists the host-port pairs running the ZooKeeper service.

这列出了多个正在运行 Zookeeper 服务的主机名-端口号信息。

> As with the parameters described earlier in the document, these settings may be configured on a per-nameservice basis by suffixing the configuration key with the nameservice ID. For example, in a cluster with federation enabled, you can explicitly enable automatic failover for only one of the nameservices by setting dfs.ha.automatic-failover.enabled.my-nameservice-id.

就像本文档中先前描述的参数一样，these settings may be configured on a per-nameservice basis by suffixing the configuration key with the nameservice ID.。例如，在开启联邦的集群中，你可能需要明确的指定所要开启自动故障转移的 nameservice，用 dfs.ha.automatic-failover.enabled.my-nameservice-id 指定。

> There are also several other configuration parameters which may be set to control the behavior of automatic failover; however, they are not necessary for most installations. Please refer to the configuration key specific documentation for details.

也有其他的配置参数，可能被用来管理自动故障转移的表现；然而，对于大多数的安装来说是不必要的。请参考特定文档获取更多信息。

### 7.6、Initializing HA state in ZooKeeper

> After the configuration keys have been added, the next step is to initialize required state in ZooKeeper. You can do so by running the following command from one of the NameNode hosts.

初始化 Zookeeper 中的必要的状态。

你可以通过在任意 NameNode 所在的主机上运行下面的命令来完成这个操作：

	[hdfs]$ $HADOOP_HOME/bin/zkfc -formatZK

> This will create a znode in ZooKeeper inside of which the automatic failover system stores its data.

这将在 Zookeeper 中创建一个 znode，自动故障转移系统存储它的数据。

### 7.7、Starting the cluster with start-dfs.sh

> Since automatic failover has been enabled in the configuration, the start-dfs.sh script will now automatically start a ZKFC daemon on any machine that runs a NameNode. When the ZKFCs start, they will automatically select one of the NameNodes to become active.

因为自动故障转移在配置中已经被开启了，start-dfs.sh 脚本将会在任意运行 NameNode 的机器上自动启动一个 ZKFC 守护进程。当 ZKFC 启动，它们将自动选择一个 NameNode 变成 Active。

### 7.8、Starting the cluster manually

> If you manually manage the services on your cluster, you will need to manually start the zkfc daemon on each of the machines that runs a NameNode. You can start the daemon by running:

如果手工管理集群上的服务，你将需要手动在将要运行 NameNode 的机器上启动 zkfc。你可以用下面的命令启动守护进程：

	[hdfs]$ $HADOOP_HOME/bin/hdfs --daemon start zkfc

### 7.9、Securing access to ZooKeeper

> If you are running a secure cluster, you will likely want to ensure that the information stored in ZooKeeper is also secured. This prevents malicious clients from modifying the metadata in ZooKeeper or potentially triggering a false failover.

如果你正在运行一个安全的集群，你很希望确保存储在 Zookeeper 中的信息也是安全的。这将防止恶意的客户端修改 Zookeeper 中的元数据，或者潜在地触发一个错误的故障转移。

> In order to secure the information in ZooKeeper, first add the following to your core-site.xml file:

为了确保 Zookeeper 中信息的安全，首先增加下面的配置到core-site.xml文件：

```xml
 <property>
   <name>ha.zookeeper.auth</name>
   <value>@/path/to/zk-auth.txt</value>
 </property>
 <property>
   <name>ha.zookeeper.acl</name>
   <value>@/path/to/zk-acl.txt</value>
 </property>
```

> Please note the ‘@’ character in these values – this specifies that the configurations are not inline, but rather point to a file on disk. The authentication info may also be read via a CredentialProvider (pls see the CredentialProviderAPI Guide in the hadoop-common project).

请注意这两个值中的 “@” 字符，这指出配置不是内联的，而是指向磁盘中的一个文件。身份验证信息将通过 CredentialProvider 读取。(pls see the CredentialProviderAPI Guide in the hadoop-common project).

> The first configured file specifies a list of ZooKeeper authentications, in the same format as used by the ZK CLI. For example, you may specify something like:

The first configured file 以跟 ZK CLI使用的相同的格式指定了一个 Zookeeper 身份验证的列表。例如，你可能像下面这样指定一些东西：

	digest:hdfs-zkfcs:mypassword

> …where hdfs-zkfcs is a unique username for ZooKeeper, and mypassword is some unique string used as a password.

…hdfs-zkfcs 是 Zookeeper 中全局唯一的用户名，mypassword 是一个唯一的字符串，作为密码。

> Next, generate a ZooKeeper ACL that corresponds to this authentication, using a command like the following:

下一步，根据这个认证，生成相应的 Zookeeper ACL，使用类似于下面的命令：

	[hdfs]$ java -cp $ZK_HOME/lib/*:$ZK_HOME/zookeeper-3.4.2.jar org.apache.zookeeper.server.auth.DigestAuthenticationProvider hdfs-zkfcs:mypassword
	output: hdfs-zkfcs:mypassword->hdfs-zkfcs:P/OQvnYyU/nF/mGYvB/xurX8dYs=

> Copy and paste the section of this output after the ‘->’ string into the file zk-acls.txt, prefixed by the string “digest:”. For example:

复制和粘贴输出 ”->“ 后的部分，写到 zk-acls.txt 文件中，要加上” digest: “前缀，例如：

	digest:hdfs-zkfcs:vlUvLnd8MlacsE80rDuu6ONESbM=:rwcda

> In order for these ACLs to take effect, you should then rerun the zkfc -formatZK command as described above.

为了使 ACL 生效，你应该重新运行 zkfc -formatZK 命令。

> After doing so, you may verify the ACLs from the ZK CLI as follows:

做了这些之后，你可能需要验证来自 ZK CLI 的 ACL，像下面这样：

	[zk: localhost:2181(CONNECTED) 1] getAcl /hadoop-ha
	'digest,'hdfs-zkfcs:vlUvLnd8MlacsE80rDuu6ONESbM=
	: cdrwa

### 7.10、Verifying automatic failover

> Once automatic failover has been set up, you should test its operation. To do so, first locate the active NameNode. You can tell which node is active by visiting the NameNode web interfaces – each node reports its HA state at the top of the page.

一旦自动故障转移被建立，你应该验证它的可用性。为了验证，首先定位 Active NameNode 。可以通过浏览 NameNode 的 web 接口分辨哪个 NameNode 是 Active 的。- 在页面尾，每个 node 报告它的 HA 状态。

> Once you have located your active NameNode, you may cause a failure on that node. For example, you can use kill -9 <pid of NN> to simulate a JVM crash. Or, you could power cycle the machine or unplug its network interface to simulate a different kind of outage. After triggering the outage you wish to test, the other NameNode should automatically become active within several seconds. The amount of time required to detect a failure and trigger a fail-over depends on the configuration of ha.zookeeper.session-timeout.ms, but defaults to 5 seconds.

一旦定位了 Active NameNode ，你可以在那个节点上造成一个故障。例如，你可以用 `kill -9 <pid of NN>` 模仿一次 JVM 崩溃。或者，你可以关闭这台机器或者拔下它的网卡来模仿一次不同类型的中断。在触发了你想测试的中断之后，另一个 NameNode 应该可以在几秒后自动转变为 Active 状态。故障检测和触发一次故障转移所需的时间依靠配置 ha.zookeeper.session-timeout.ms 来实现，默认是5秒。

> If the test does not succeed, you may have a misconfiguration. Check the logs for the zkfc daemons as well as the NameNode daemons in order to further diagnose the issue.

如果检测不成功，你可能丢掉了配置，检查 zkfc 和 NameNode 进程的日志文件以进行进一步的问题检测。

## 8、Automatic Failover FAQ

> Is it important that I start the ZKFC and NameNode daemons in any particular order?

> No. On any given node you may start the ZKFC before or after its corresponding NameNode.

(1)我启动 ZKFC 和 NameNode 守护进程的顺序重要么？

	不重要，在任何给定的节点上，你可以任意顺序启动 ZKFC 和 NameNode 进程。

> What additional monitoring should I put in place?

> You should add monitoring on each host that runs a NameNode to ensure that the ZKFC remains running. In some types of ZooKeeper failures, for example, the ZKFC may unexpectedly exit, and should be restarted to ensure that the system is ready for automatic failover.

> Additionally, you should monitor each of the servers in the ZooKeeper quorum. If ZooKeeper crashes, then automatic failover will not function.

(2)我应该在此增加什么样的监控？

	你应该在每一台运行 NameNode 的机器上增加监控以确保 ZKFC 保持运行。在某些类型的
	Zookeeper 失效时，例如，ZKFC 意料之外的结束，应该被重新启动以确保，系统准备自动故障转移。

	除此之外，你应该监控 Zookeeper 集群中的每一个 Server。如果 Zookeeper 宕机，
	自动故障转移将不起作用。

> What happens if ZooKeeper goes down?

> If the ZooKeeper cluster crashes, no automatic failovers will be triggered. However, HDFS will continue to run without any impact. When ZooKeeper is restarted, HDFS will reconnect with no issues.

(3)如果Zookeeper宕机会怎样？

	如果 Zookeeper 集群宕机，没有自动故障转移将会被触发。但是，HDFS 将继续没有任何影响的运行。
	当 Zookeeper 被重新启动，HDFS 将重新连接，不会出现问题。

> Can I designate one of my NameNodes as primary/preferred?

> No. Currently, this is not supported. Whichever NameNode is started first will become active. You may choose to start the cluster in a specific order such that your preferred node starts first.

(4)我可以指定两个NameNode中的一个作为主要的NameNode么？

	当然不可以。目前，这是不支持的。先启动的 NameNode 将会先变成 Active
	 状态。你可以特定的顺序，先启动你希望成为 Active 的节点，来完成这个目的。

> How can I initiate a manual failover when automatic failover is configured?

> Even if automatic failover is configured, you may initiate a manual failover using the same hdfs haadmin command. It will perform a coordinated failover.

(5)自动故障转移被配置时，如何发起一次手工故障转移？

	即使自动故障转移被卑职，你也可以用 hdfs haadmin
	 发起一次手工故障转移。这个命令将执行一次协调的故障转移。