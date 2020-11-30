# MR Support for YARN Shared Cache

[TOC]

## 1、Overview

> MapReduce support for the YARN shared cache allows MapReduce jobs to take advantage of additional resource caching. This saves network bandwidth between the job submission client as well as within the YARN cluster itself. This will reduce job submission time and overall job runtime.

MapReduce 支持 YARN 共享缓存，允许 MapReduce jobs 利用额外的资源进行缓存。

这节省了提交 job 的客户端和 YARN 集群本间的网络带宽。这将减少 job 提交时间和整个 job 运行时长。

## 2、Enabling/Disabling the shared cache

> First, your YARN cluster must have the shared cache service running. Please see YARN documentation for information on how to setup the shared cache service.

首先，YARN 集群必须运行共享缓存服务。

有关如何设置共享缓存服务的信息，请参阅 YARN 文档。

> A MapReduce user can specify what resources are eligible to be uploaded to the shared cache based on resource type. This is done using a configuration parameter in mapred-site.xml:

**MapReduce 用户可以根据资源类型指定哪些资源可以上传到共享缓存中**。

这是通过 `map-site.xml` 中的配置参数来完成的:

```xml
<property>
    <name>mapreduce.job.sharedcache.mode</name>
    <value>disabled</value>
    <description>
       A comma delimited list of resource categories to submit to the
       shared cache. The valid categories are: jobjar, libjars, files,
       archives. If "disabled" is specified then the job submission code will not use the shared cache.
       提交到共享缓存的资源目录，用逗号分隔。如jobjar, libjars, files
       如果设置了disabled，就不会使用共享缓存
    </description>
</property>
```
如果列出了资源类型，它将检查共享缓存，以查看该资源是否已经在缓存中。如果是，它将使用缓存的资源，如果不是，需要异步上传资源。

> If a resource type is listed, it will check the shared cache to see if the resource is already in the cache. If so, it will use the cached resource, if not, it will specify that the resource needs to be uploaded asynchronously.

## 3、Specifying resources for the cache

> A MapReduce user has 3 ways to specify resources for a MapReduce job:

用户有三种方法为 MapReduce job 指定资源:

> 1.The command line via the generic options parser (i.e. -files, -archives, -libjars): If a resource is specified via the command line and the resource type is enabled for the shared cache, that resource will use the shared cache.

- 通用选项解析器的命令行(如，`-files`、`-archives`、`-libjar`)：

	如果通过命令行指定了资源，并且为共享缓存启用了资源类型，那么该资源将使用共享缓存。

> 2.The distributed cache api: If a resource is specified via the distributed cache the resource will not use the shared cache regardless of if the resource type is enabled for the shared cache.

- 分布式缓存api：

	如果资源是通过分布式缓存指定的，那么无论资源类型是否为共享缓存启用，该资源都不会使用共享缓存。

> 3.The shared cache api: This is a new set of methods added to the org.apache.hadoop.mapreduce.Job api. It allows users to add a file to the shared cache, add it to the shared cache and the classpath and add an archive to the shared cache. These resources will be placed in the distributed cache and, if their resource type is enabled the client will use the shared cache as well.

- 共享缓存api：

	这是添加到 `org.apache.hadoop.mapreduce.Job` 的一组新方法。

	允许用户将文件添加到共享缓存，将其添加到共享缓存和 classpath ，并将 archive 添加到共享缓存。

	这些资源将被放置在分布式缓存中，如果启用了它们的资源类型，客户端也将使用共享缓存。

## 4、Resource naming

> It is important to ensure that each resource for a MapReduce job has a unique file name. **This prevents symlink clobbering when YARN containers running MapReduce tasks are localized during container launch.** A user can specify their own resource name by using the fragment portion of a URI. For example, for file resources specified on the command line, it could look like this:

**确保 MapReduce job 的每个资源有唯一的文件名**是很重要的。

**用户可以使用 URI 的片段部分来指定它们自己的资源名**。

例如，对于命令行指定的文件资源，它可能是这样的:

	-files /local/path/file1.txt#foo.txt,/local/path2/file1.txt#bar.txt

在上面的例子中，名为`file1.txt`的两个文件，将本地化为两个不同的名称:`foo.txt`和`bar.txt`。

> In the above example two files, named file1.txt, will be localized with two different names: foo.txt and bar.txt.

## 5、Resource Visibility

> All resources in the shared cache have a PUBLIC visibility.

共享缓存中的所有资源都是 PUBLIC 的。

## 6、MapReduce client behavior while the shared cache is unavailable

> In the event that the shared cache manager is unavailable, the MapReduce client uses a fail-fast mechanism. If the MapReduce client fails to contact the shared cache manager, the client will no longer use the shared cache for the rest of that job submission. This prevents the MapReduce client from timing out each time it tries to check for a resource in the shared cache. The MapReduce client quickly reverts to the default behavior and submits a Job as if the shared cache was never enabled in the first place.

在`共享缓存管理器`不可用的情况下，MapReduce 客户端使用一种快速失败机制。

如果 MapReduce 客户端未能与共享缓存管理器联系，那么对于 job 提交的剩余部分，客户端将不再使用共享缓存。

这可以防止 MapReduce 客户端在每次检查共享缓存中的资源时超时。

MapReduce 客户端会迅速恢复到默认的行为，并提交一个 Job，就好像共享缓存一开始就没有启用过一样。