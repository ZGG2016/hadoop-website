# YARN Shared Cache

[TOC]

## 1、Overview

> The YARN Shared Cache provides the facility to upload and manage shared application resources to HDFS in a safe and scalable manner. YARN applications can leverage resources uploaded by other applications or previous runs of the same application without having to re­upload and localize identical files multiple times. This will save network resources and reduce YARN application startup time.

YARN 共享缓存提供了，以安全和可扩展的方式，上传和管理共享应用程序资源到 HDFS 的功能。

**YARN 应用程序可以利用其他应用程序或以前运行同一应用程序时上传的资源，而不必多次重新上传和本地化相同的文件**。

这将节省网络资源，减少 YARN 应用程序启动时间。

## 2、Current Status and Future Plans

> Currently the YARN Shared Cache is released and ready to use. The major components are implemented and have been deployed in a large-scale production setting. There are still some pieces missing (i.e. strong authentication). These missing features will be implemented as part of a follow-up phase 2 effort. Please see YARN-7282 for more information.

目前，YARN 共享缓存已经释放，准备使用。

主要组件已在大规模生产环境中实现并部署。仍然有一些缺失(如强身份验证)。

这些缺失的特性将作为后续阶段2工作的一部分。更多信息请见 YARN-7282。

## 3、Architecture

> The shared cache feature consists of 4 major components:

**共享缓存特征由4个主要部分组成：**

> The shared cache client.

1.共享缓存客户端。

> The HDFS directory that acts as a cache.

2.作为缓存的 HDFS 目录。

> The shared cache manager (aka. SCM).

3.共享缓存管理器(SCM)。

> The localization service and uploader.

4.本地化服务和上传器。

### 3.1、The Shared Cache Client

> YARN application developers and users, should interact with the shared cache using the shared cache client. This client is responsible for interacting with the shared cache manager, computing the checksum of application resources, and claiming application resources in the shared cache. Once an application has claimed a resource, it is free to use that resource for the life-cycle of the application. Please see the SharedCacheClient.java javadoc for further documentation.

YARN 应用程序开发人员和用户，应该使用共享缓存客户端与共享缓存交互。

**此客户端负责与共享缓存管理器交互，计算应用程序资源的校验和，并在共享缓存中声明应用程序资源**。

一旦应用程序声明了资源，它就可以在应用程序的生命周期中自由地使用该资源。

请参阅 `SharedCacheClient.java javadoc` 获取更多文档。

### 3.2、The Shared Cache HDFS Directory

> The shared cache HDFS directory stores all of the shared cache resources. It is protected by HDFS permissions and is globally readable, but writing is restricted to a trusted user. This HDFS directory is only modified by the shared cache manager and the resource uploader on the node manager. Resources are spread across a set of subdirectories using the resources’s checksum:

**共享缓存 HDFS 目录存储所有共享缓存资源。它受 HDFS 权限保护，是全局可读的，但写入仅限于可信的用户**。

这个 HDFS 目录**仅由共享缓存管理器和节点管理器上的资源上传器修改**。

使用资源的校验和，资源分布在一组子目录上:

	/sharedcache/a/8/9/a896857d078/foo.jar
	/sharedcache/5/0/f/50f11b09f87/bar.jar
	/sharedcache/a/6/7/a678cb1aa8f/job.jar

### 3.3、Shared Cache Manager (SCM)

> The shared cache manager is responsible for serving requests from the client and managing the contents of the shared cache. It looks after both the meta data as well as the persisted resources in HDFS. It is made up of two major components, a back end store and a cleaner service. The SCM runs as a separate daemon process that can be placed on any node in the cluster. This allows for administrators to start/stop/upgrade the SCM without affecting other YARN components (i.e. the resource manager or node managers).

**共享缓存管理器负责为来自客户端的请求提供服务，并管理共享缓存的内容**。

它**既负责处理元数据，也负责处理 HDFS 中的持久化资源。它由两个主要组件组成，后端存储和清洁服务**。

SCM 作为单独的守护进程运行，可以运行在集群中的任何节点上。这就允许管理员启动/停止/升级SCM，而不影响其他 YARN 组件(如resource manager 或 node managers)。

> The back end store is responsible for maintaining and persisting metadata about the shared cache. This includes the resources in the cache, when a resource was last used and a list of applications that are currently using the resource. The implementation for the backing store is pluggable and it currently uses an in-memory store that recreates its state after a restart.

**后端存储负责维护和持久化关于共享缓存的元数据**。

这包括缓存中的资源、最后一次使用资源的时间以及当前使用该资源的应用程序列表。

后备存储的实现是可插拔的，它目前使用一个在重启后重新创建其状态的内存存储。

> The cleaner service maintains the persisted resources in HDFS by ensuring that resources that are no longer used are removed from the cache. It scans the resources in the cache periodically and evicts resources if they are both stale and there are no live applications currently using the application.

**通过确保从缓存中删除不再使用的资源，清洁服务以此来维护 HDFS 中的持久资源**。

它定期扫描缓存中的资源，如果资源已经过时，并且当前没有活跃的应用程序使用，则清除这些资源。

### 3.4、The Shared Cache uploader and localization

> The shared cache uploader is a service that runs on the node manager and adds resources to the shared cache. It is responsible for verifying a resources checksum, uploading the resource to HDFS and notifying the shared cache manager that a resource has been added to the cache. It is important to note that the uploader service is asynchronous from the container launch and does not block the startup of a yarn application. In addition adding things to the cache is done in a best effort way and does not impact running applications. Once the uploader has placed a resource in the shared cache, YARN uses the normal node manager localization mechanism to make resources available to the application.

**共享缓存上传器是运行在 node manager 上并向共享缓存添加资源的服务**。

它负责验证资源校验和，将资源上传到 HDFS，并通知共享缓存管理器一个资源已经被添加到缓存中。

上传器服务在 container 启动时是异步的，并且不会阻止 yarn 应用程序的启动，注意这一点很重要。

此外，向缓存添加内容是用最好的方式完成的，并且不会影响运行的应用程序。一旦上传者在共享缓存中放置了资源，YARN 使用普通的 node manager 本地化机制使资源对应用程序可用。

## 4、Developing YARN applications with the Shared Cache

> To support the YARN shared cache, an application must use the shared cache client during application submission. The shared cache client returns a URL corresponding to a resource if it is in the shared cache. To use the cached resource, a YARN application simply uses the cached URL to create a LocalResource object and sets setShouldBeUploadedToSharedCache to true during application submission.

为了支持 YARN 共享缓存，在提交应用程序时，应用程序必须使用共享缓存客户端。

共享缓存客户端返回一个与资源对应的 URL，如果资源在共享缓存中。

为了使用缓存的资源，YARN 应用程序只需使用缓存的 URL 创建 LocalResource 对象，并在应用程序提交期间将 `setShouldBeUploadedToSharedCache`设置为 true。

> For example, here is how you would create a LocalResource using a cached URL:

	String localPathChecksum = sharedCacheClient.getFileChecksum(localPath);
	URL cachedResource = sharedCacheClient.use(appId, localPathChecksum);
	LocalResource resource = LocalResource.newInstance(cachedResource,
	      LocalResourceType.FILE, LocalResourceVisibility.PUBLIC
	      size, timestamp, null, true);

## 5、Administrating the Shared Cache

### 5.1、Setting up the shared cache

> An administrator can initially set up the shared cache by following these steps:

管理员可以通过以下步骤设置共享缓存:

> Create an HDFS directory for the shared cache (default: /sharedcache).

1.为共享缓存创建一个 HDFS 目录(默认为:`/sharedcache`)。

> Set the shared cache directory permissions to 0755.

2.将共享缓存目录权限设置为 0755。

> Ensure that the shared cache directory is owned by the user that runs the shared cache manager daemon and the node manager.

3.确保共享缓存目录为运行共享缓存管理器守护进程和 node manager 的用户所拥有。

> In the yarn-site.xml file, set yarn.sharedcache.enabled to true and yarn.sharedcache.root-dir to the directory specified in step 1. For more configuration parameters, see the configuration parameters section.

4.在 `yarn-site.xml` 文件中，设置 `yarn.sharedcache.enabled` 为 true和 `yarn.sharedcache.root-dir` 为步骤1中指定的目录。

> Start the shared cache manager:

5.启动共享缓存管理器：

	/hadoop/bin/yarn --daemon start sharedcachemanager

### 5.2、Configuration parameters

> The configuration parameters can be found in yarn-default.xml and should be set in the yarn-site.xml file. Here are a list of configuration parameters and their defaults:

配置参数可以在 `yarn-default.xml` 中找到，并设置。

Name | Description | Default value
---|:---|:---
yarn.sharedcache.enabled  | Whether the shared cache is enabled【是否启动共享缓存】 | false
yarn.sharedcache.root-dir | The root directory for the shared cache【共享缓存的根目录】 | /sharedcache
yarn.sharedcache.nested-level | The level of nested directories before getting to the checksum directories. It must be non-negative.【在到达校验和目录之前，嵌套目录级别。它一定是非负的。】 | 3
yarn.sharedcache.store.class | The implementation to be used for the SCM store【用于SCM存储的实现】 | org.apache.hadoop.yarn.server.sharedcachemanager.store.InMemorySCMStore
yarn.sharedcache.app-checker.class | The implementation to be used for the SCM app-checker【用于SCM app-checker的实现】 | org.apache.hadoop.yarn.server.sharedcachemanager.RemoteAppChecker
yarn.sharedcache.store.in-memory.staleness-period-mins | A resource in the in-memory store is considered stale if the time since the last reference exceeds the staleness period. This value is specified in minutes.【如果自上次引用以来的时间超过了期限，则认为内存存储中的资源已过期。该值以分钟为单位】 |	10080
yarn.sharedcache.store.in-memory.initial-delay-mins | Initial delay before the in-memory store runs its first check to remove dead initial applications. Specified in minutes.【在内存存储运行第一次检查以移除死的初始应用程序之前的初始延迟。单位为分钟。】 | 10
yarn.sharedcache.store.in-memory.check-period-mins | The frequency at which the in-memory store checks to remove dead initial applications. Specified in minutes. 【内存存储检查以删除死的初始应用程序的频率。单位为分钟。】| 720
yarn.sharedcache.admin.address | The address of the admin interface in the SCM (shared cache manager)【SCM中管理员接口的地址】 | 0.0.0.0:8047
yarn.sharedcache.admin.thread-count | The number of threads used to handle SCM admin interface (1 by default)【用来处理SCM管理员接口的线程数】 | 1
yarn.sharedcache.webapp.address | The address of the web application in the SCM (shared cache manager) 【SCM中web应用程序的地址】| 0.0.0.0:8788
yarn.sharedcache.cleaner.period-mins | The frequency at which a cleaner task runs. Specified in minutes. 【清洁器任务运行的频率。单位为分钟。】| 1440
yarn.sharedcache.cleaner.initial-delay-mins | Initial delay before the first cleaner task is scheduled. Specified in minutes.【在第一个清洁器任务被调度前的初始延迟。单位为分钟。】 | 10
yarn.sharedcache.cleaner.resource-sleep-ms | The time to sleep between processing each shared cache resource. Specified in milliseconds. 【处理两个共享缓存资源间的睡眠时间】| 0
yarn.sharedcache.uploader.server.address | The address of the node manager interface in the SCM (shared cache manager) 【SCM中node manager接口的地址】| 0.0.0.0:8046
yarn.sharedcache.uploader.server.thread-count | The number of threads used to handle shared cache manager requests from the node manager (50 by default)【用来处理来自node manager的共享缓存管理器请求的线程数】 | 50
yarn.sharedcache.client-server.address | The address of the client interface in the SCM (shared cache manager)【SCM中的客户端接口的地址】| 0.0.0.0:8045
yarn.sharedcache.client-server.thread-count | The number of threads used to handle shared cache manager requests from clients (50 by default)【用来处理来自客户端的共享缓存管理器请求的线程数】 | 50
yarn.sharedcache.checksum.algo.impl | The algorithm used to compute checksums of files (SHA-256 by default)【用来计算文件校验和的算法】 | org.apache.hadoop.yarn.sharedcache.ChecksumSHA256Impl
yarn.sharedcache.nm.uploader.replication.factor | The replication factor for the node manager uploader for the shared cache (10 by default)【共享缓存的nodemanager上传器的复制因子】 | 10
yarn.sharedcache.nm.uploader.thread-count | The number of threads used to upload files from a node manager instance (20 by default) 【用于从node manager实例上传文件的线程数】| 20