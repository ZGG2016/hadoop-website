# ResourceManager High Availability

[TOC]

## 1、Introduction

> This guide provides an overview of High Availability of YARN’s ResourceManager, and details how to configure and use this feature. The ResourceManager (RM) is responsible for tracking the resources in a cluster, and scheduling applications (e.g., MapReduce jobs). Prior to Hadoop 2.4, the ResourceManager is the single point of failure in a YARN cluster. The High Availability feature adds redundancy in the form of an Active/Standby ResourceManager pair to remove this otherwise single point of failure.

本指南提供了关于 YARN ResourceManager(RM) 高可用的概览，以及如何配置和使用这个特性。**RM 复制追踪集群中的资源，调度应用程序(如，MapReduce jobs)**。

在 Hadoop 2.4 版本之前，RM 在 YARN 集群中具有单点故障。高可用特性以 Active/Standby RM 对的形式增加了冗余，来删除单点故障。

## 2、Architecture

![rm-ha-overview](./image/rm-ha-overview.png)

### 2.1、RM Failover

> ResourceManager HA is realized through an Active/Standby architecture - at any point of time, one of the RMs is Active, and one or more RMs are in Standby mode waiting to take over should anything happen to the Active. The trigger to transition-to-active comes from either the admin (through CLI) or through the integrated failover-controller when automatic-failover is enabled.

**RM 的高可用通过一个 Active/Standby 架构实现 -- 在任意时刻，只有一个 RM 是 Active，其他的一个或多个是 Standby，在 Active 故障时，替代它**。

转成 Active 的触发机制要么来管理员(通过CLI)，要么集成的 failover-controller，当自动 failover 启用时。

#### 2.1.1、Manual transitions and failover

> When automatic failover is not enabled, admins have to manually transition one of the RMs to Active. To failover from one RM to the other, they are expected to first transition the Active-RM to Standby and transition a Standby-RM to Active. All this can be done using the “yarn rmadmin” CLI.

**当自动 failover 没有启用时，管理员必须手动地转换一个 RM 为 Active**。为了将 failover 从一个 RM 转到另一个，需要首先将 Active-RM 转成 Standby，然后将一个 Standby-RM 转成 Active。所有的这些**可以使用 "yarn rmadmin" CLI 完成**。

#### 2.1.2、Automatic failover

> The RMs have an option to embed the Zookeeper-based ActiveStandbyElector to decide which RM should be the Active. When the Active goes down or becomes unresponsive, another RM is automatically elected to be the Active which then takes over. Note that, there is no need to run a separate ZKFC daemon as is the case for HDFS because ActiveStandbyElector embedded in RMs acts as a failure detector and a leader elector instead of a separate ZKFC deamon.

**RMs 有一个选项来集成基于 Zookeeper 的 ActiveStandbyElector，这个的作用就是决定哪个 RM 应当成为 Active**。当 Active RM 故障，其他 RM 自动被选择为 Active。

注意：**不需要像 HDFS 那样运行一个单独的 ZKFC 进程**，因为集成在 RMs 的 ActiveStandbyElector 的行为就是一个故障检测器和一个 leader elector，而不需采用一个单独的 ZKFC 进程

#### 2.1.3、Client, ApplicationMaster and NodeManager on RM failover

> When there are multiple RMs, the configuration (yarn-site.xml) used by clients and nodes is expected to list all the RMs. Clients, ApplicationMasters (AMs) and NodeManagers (NMs) try connecting to the RMs in a round-robin fashion until they hit the Active RM. If the Active goes down, they resume the round-robin polling until they hit the “new” Active. This default retry logic is implemented as org.apache.hadoop.yarn.client.ConfiguredRMFailoverProxyProvider. You can override the logic by implementing org.apache.hadoop.yarn.client.RMFailoverProxyProvider and setting the value of yarn.client.failover-proxy-provider to the class name.

当有多个 RMs 时，客户端和节点使用的配置文件(yarn-site.xml)将列出所有的 RMs。**客户端、 ApplicationMasters(AMs) 和 NodeManagers (NMs) 会以轮询的方式连接 RMs，直到它们连接到 Active RM**。如果 Active RM 故障，它们将继续循环轮询，直到连接到新的 Active RM。 

这个**默认的重试机制是由 `org.apache.hadoop.yarn.client.ConfiguredRMFailoverProxyProvider` 实现。可以通过实现它来重写这个逻辑，然后设置 `yarn.client.failover-proxy-provider` 的值为这个类**。

### 2.2、Recovering previous active-RM’s state

> With the [ResourceManager Restart](https://hadoop.apache.org/docs/r3.2.1/hadoop-yarn/hadoop-yarn-site/ResourceManagerRestart.html) enabled, the RM being promoted to an active state loads the RM internal state and continues to operate from where the previous active left off as much as possible depending on the RM restart feature. A new attempt is spawned for each managed application previously submitted to the RM. Applications can checkpoint periodically to avoid losing any work. The state-store must be visible from the both of Active/Standby RMs. Currently, there are two RMStateStore implementations for persistence - FileSystemRMStateStore and ZKRMStateStore. The ZKRMStateStore implicitly allows write access to a single RM at any point in time, and hence is the recommended store to use in an HA cluster. When using the ZKRMStateStore, there is no need for a separate fencing mechanism to address a potential split-brain situation where multiple RMs can potentially assume the Active role. When using the ZKRMStateStore, it is advisable to NOT set the “zookeeper.DigestAuthenticationProvider.superDigest” property on the Zookeeper cluster to ensure that the zookeeper admin does not have access to YARN application/user credential information.

启用了 ResourceManager 重启后，提升为活动状态的 RM 将加载 RM 内部状态，并根据 RM 重启特性，尽可能从前一活动状态停止的位置继续操作。

将为以前提交给 RM 的每个托管应用程序生成一个新的尝试。应用程序可以定期 checkpoint 以避免丢失任何工作。

状态存储必须对 Active/Standby 都可见。目前，**有两种 RMStateStore 持久化实现 -- FileSystemRMStateStore 和 ZKRMStateStore**。

ZKRMStateStore 隐式地允许在任何时间点对单个 RM 进行的写访问，因此是建议在 HA 集群中使用的存储。

在使用 ZKRMStateStore 时，不需要单独的隔离机制，来解决潜在的 split-brain 情况，在这种情况下，多个 RM 可能会扮演 Active 角色。

当使用 ZKRMStateStore 时，建议不要在 Zookeeper 集群上设置 `zookeeper.DigestAuthenticationProvider.superDigest` 属性，以确保 Zookeeper 管理员无权访问 YARN 应用程序/用户凭据信息。

## 3、Deployment

> Most of the failover functionality is tunable using various configuration properties. Following is a list of required/important ones. yarn-default.xml carries a full-list of knobs. See [yarn-default.xml](https://hadoop.apache.org/docs/r3.2.1/hadoop-yarn/hadoop-yarn-common/yarn-default.xml) for more information including default values. See the document for [ResourceManager Restart](https://hadoop.apache.org/docs/r3.2.1/hadoop-yarn/hadoop-yarn-site/ResourceManagerRestart.html) also for instructions on setting up the state-store.

大多数 failover 功能都可以使用各种配置属性进行调优。

以下是必需/重要事项的列表。`yarn-default.xml` 包含一个完整的旋钮列表。有关包括默认值的更多信息，请参见 `yarn-default.xml`。请参阅有关 ResourceManager Restart 的文档，也可以了解关于设置状态存储的说明。

Configuration Properties   |   Description
---|:---
hadoop.zk.address   |   Address of the ZK-quorum. Used both for the state-store and embedded leader-election.
yarn.resourcemanager.ha.enabled   |   Enable RM HA.
yarn.resourcemanager.ha.rm-ids    |   List of logical IDs for the RMs. e.g., “rm1,rm2”.
yarn.resourcemanager.hostname.rm-id   |   For each rm-id, specify the hostname the RM corresponds to. Alternately, one could set each of the RM’s service addresses.
yarn.resourcemanager.address.rm-id    |   For each rm-id, specify host:port for clients to submit jobs. If set, overrides the hostname set in yarn.resourcemanager.hostname.rm-id.
yarn.resourcemanager.scheduler.address.rm-id   |   For each rm-id, specify scheduler host:port for ApplicationMasters to obtain resources. If set, overrides the hostname set in yarn.resourcemanager.hostname.rm-id.
yarn.resourcemanager.resource-tracker.address.rm-id   |   For each rm-id, specify host:port for NodeManagers to connect. If set, overrides the hostname set in yarn.resourcemanager.hostname.rm-id.
yarn.resourcemanager.admin.address.rm-id   |   For each rm-id, specify host:port for administrative commands. If set, overrides the hostname set in yarn.resourcemanager.hostname.rm-id.
yarn.resourcemanager.webapp.address.rm-id   |   For each rm-id, specify host:port of the RM web application corresponds to. You do not need this if you set yarn.http.policy to HTTPS_ONLY. If set, overrides the hostname set in yarn.resourcemanager.hostname.rm-id.
yarn.resourcemanager.webapp.https.address.rm-id   |   For each rm-id, specify host:port of the RM https web application corresponds to. You do not need this if you set yarn.http.policy to HTTP_ONLY. If set, overrides the hostname set in yarn.resourcemanager.hostname.rm-id.
yarn.resourcemanager.ha.id   |   Identifies the RM in the ensemble. This is optional; however, if set, admins have to ensure that all the RMs have their own IDs in the config.
yarn.resourcemanager.ha.automatic-failover.enabled   |   Enable automatic failover; By default, it is enabled only when HA is enabled.
yarn.resourcemanager.ha.automatic-failover.embedded   |   Use embedded leader-elector to pick the Active RM, when automatic failover is enabled. By default, it is enabled only when HA is enabled.
yarn.resourcemanager.cluster-id   |   Identifies the cluster. Used by the elector to ensure an RM doesn’t take over as Active for another cluster.
yarn.client.failover-proxy-provider   |   The class to be used by Clients, AMs and NMs to failover to the Active RM.
yarn.client.failover-max-attempts   |   The max number of times FailoverProxyProvider should attempt failover.
yarn.client.failover-sleep-base-ms   |   The sleep base (in milliseconds) to be used for calculating the exponential delay between failovers.
yarn.client.failover-sleep-max-ms   |   The maximum sleep time (in milliseconds) between failovers.
yarn.client.failover-retries   |   The number of retries per attempt to connect to a ResourceManager.
yarn.client.failover-retries-on-socket-timeouts   |   The number of retries per attempt to connect to a ResourceManager on socket timeouts.

### 3.1、Configurations

> Here is the sample of minimal setup for RM failover.

```xml
<property>
	<name>yarn.resourcemanager.ha.enabled</name>
	<value>true</value>
</property>
<property>
	<name>yarn.resourcemanager.cluster-id</name>
	<value>cluster1</value>
</property>
<property>
	<name>yarn.resourcemanager.ha.rm-ids</name>
	<value>rm1,rm2</value>
</property>
<property>
	<name>yarn.resourcemanager.hostname.rm1</name>
	<value>master1</value>
</property>
<property>
	<name>yarn.resourcemanager.hostname.rm2</name>
	<value>master2</value>
</property>
<property>
	<name>yarn.resourcemanager.webapp.address.rm1</name>
	<value>master1:8088</value>
</property>
<property>
	<name>yarn.resourcemanager.webapp.address.rm2</name>
	<value>master2:8088</value>
</property>
<property>
	<name>hadoop.zk.address</name>
	<value>zk1:2181,zk2:2181,zk3:2181</value>
</property>
```

### 3.2、Admin commands

> yarn rmadmin has a few HA-specific command options to check the health/state of an RM, and transition to Active/Standby. Commands for HA take service id of RM set by yarn.resourcemanager.ha.rm-ids as argument.

`yarn rmadmin` 是检查一个 RM 的健康/状态 和转换 Active/Standby 的 HA 专用的命令。命令接收在 `yarn.resourcemanager.ha.rm-ids` 中设置的 RM id。

```sh
 $ yarn rmadmin -getServiceState rm1
 active
 
 $ yarn rmadmin -getServiceState rm2
 standby
```

> If automatic failover is enabled, you can not use manual transition command. Though you can override this by –forcemanual flag, you need caution.

如果启用了自动 failover，你不能手动转换命令。

虽然你可以用强制的方式覆盖这个标志，但是你需要小心。

```sh
 $ yarn rmadmin -transitionToStandby rm1
 Automatic failover is enabled for org.apache.hadoop.yarn.client.RMHAServiceTarget@1d8299fd
 Refusing to manually manage HA state, since it may cause
 a split-brain scenario or other incorrect state.
 If you are very sure you know what you are doing, please
 specify the forcemanual flag.
```

> See [YarnCommands](https://hadoop.apache.org/docs/r3.2.1/hadoop-yarn/hadoop-yarn-site/YarnCommands.html) for more details.

### 3.3、ResourceManager Web UI services

> Assuming a standby RM is up and running, the Standby automatically redirects all web requests to the Active, except for the “About” page.

假定一个备份 RM 启动并运行，Standby RM 会自动将所有 web 请求重定向到 Active，“About” 页面除外。

### 3.4、Web Services

> Assuming a standby RM is up and running, RM web-services described at ResourceManager REST APIs when invoked on a standby RM are automatically redirected to the Active RM.

假设备用 RM 启动并运行，当在备用 RM 上调用 ResourceManager REST APIs 中描述的 RM web 服务将自动重定向到 Active RM。

### 3.5、Load Balancer Setup

> If you are running a set of ResourceManagers behind a Load Balancer (e.g. [Azure](https://docs.microsoft.com/en-us/azure/load-balancer/load-balancer-custom-probe-overview) or [AWS](https://docs.aws.amazon.com/elasticloadbalancing/latest/classic/elb-healthchecks.html) ) and would like the Load Balancer to point to the active RM, you can use the /isActive HTTP endpoint as a health probe. http://RM_HOSTNAME/isActive will return a 200 status code response if the RM is in Active HA State, 405 otherwise.

如果你在 Load Balancer(如Azure或AWS)后面运行一组 ResourceManagers ，并希望 Load Balancer 指向 active RM，你可以使用 `/isActive HTTP` 端点作为运行状况探测。如果 RM 处于 Active HA 状态，`http://RM_HOSTNAME/isActive` 将返回一个 200 状态码响应，否则将返回 405。