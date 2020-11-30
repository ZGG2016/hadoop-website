# Hadoop: Distributed Cache Deploy

[TOC]

## 1、Introduction

> The MapReduce application framework has rudimentary support for deploying a new version of the MapReduce framework via the distributed cache. By setting the appropriate configuration properties, users can run a different version of MapReduce than the one initially deployed to the cluster. For example, cluster administrators can place multiple versions of MapReduce in HDFS and configure mapred-site.xml to specify which version jobs will use by default. This allows the administrators to perform a rolling upgrade of the MapReduce framework under certain conditions.

MapReduce 应用程序框架支持**通过分布式缓存部署新版本的 MapReduce 框架**。

通过设置适当的配置属性，用户可以运行与最初部署到集群的版本不同的 MapReduce 版本。

例如，**集群管理员可以在 HDFS 中放置多个版本的 MapReduce，并配置 `mapred-site.xml`来指定 jobs 默认使用哪个版本**。

这**允许管理员在特定条件下滚动升级 MapReduce 框架**。

## 2、Preconditions and Limitations

> The support for deploying the MapReduce framework via the distributed cache currently does not address the job client code used to submit and query jobs. It also does not address the ShuffleHandler code that runs as an auxilliary service within each NodeManager. As a result the following limitations apply to MapReduce versions that can be successfully deployed via the distributed cache in a rolling upgrade fashion:

目前，**通过对分布式缓存部署 MapReduce 框架的支持不会处理用于提交和查询 jobs 的 job 客户端代码**。

**它也不会处理每个 NodeManager 中作为辅助服务运行的 ShuffleHandler 代码**。

因此，**以下限制适用于可以以滚动升级的方式，通过分布式缓存成功部署的MapReduce 版本**:

> The MapReduce version must be compatible with the job client code used to submit and query jobs. If it is incompatible then the job client must be upgraded separately on any node from which jobs using the new MapReduce version will be submitted or queried.

- MapReduce 版本必须**与用于提交和查询 jobs 的 jobs 客户端代码兼容**。

【如果不兼容，那么 jobs 客户端必须在使用新 MapReduce 版本提交或查询是 jobs 的任意节点上单独升级。】

> The MapReduce version must be compatible with the configuration files used by the job client submitting the jobs. If it is incompatible with that configuration (e.g.: a new property must be set or an existing property value changed) then the configuration must be updated first.

- MapReduce 版本必须**与提交 jobs 的 job 客户端所使用的配置文件兼容**。

如果它与那个配置不兼容(例如:必须设置一个新的属性或更改一个现有的属性值)，那么配置必须首先更新。

> The MapReduce version must be compatible with the ShuffleHandler version running on the nodes in the cluster. If it is incompatible then the new ShuffleHandler code must be deployed to all the nodes in the cluster, and the NodeManagers must be restarted to pick up the new ShuffleHandler code.

- MapReduce 版本必须**与运行在集群节点上的 ShuffleHandler 版本兼容**。

如果不兼容，则必须将新的 ShuffleHandler 代码部署到集群中的所有节点，并且必须重新启动 nodemanager 以获取新的 ShuffleHandler 代码。

## 3、Deploying a New MapReduce Version via the Distributed Cache

> Deploying a new MapReduce version consists of three steps:

**通过如下三个步骤部署一个新的 MapReduce 版本**：

> 1.Upload the MapReduce archive to a location that can be accessed by the job submission client. Ideally the archive should be on the cluster’s default filesystem at a publicly-readable path. See the archive location discussion below for more details. You can use the framework uploader tool to perform this step like mapred frameworkuploader -target hdfs:///mapred/framework/hadoop-mapreduce-3.2.1.tar#mrframework. It will select the jar files that are in the classpath and put them into a tar archive specified by the -target and -fs options. The tool then returns a suggestion of how to set mapreduce.application.framework.path and mapreduce.application.classpath.

- 1.**将 MapReduce archive 上传到一个可以被 job 提交客户端访问到的位置**。

	理想情况下，archive 应当在集群默认文件系统的公共可读取的路径上。

	你可以使用框架上传工具，来执行这一步骤，如`mapred frameworkuploader -target hdfs:///mapred/framework/hadoop-mapreduce-3.2.1.tar#mrframework`。

	它将选择 classpath 中的 jar 文件，通过 `-target` 和 `-fs`，将它们放入一个 tar 包。然后工具就会返回一个如何设置`mapreduce.application.framework.path`和`mapreduce.application.classpath`的建议。

	-fs：目标文件系统。默认是通过`fs.defaultFS`设置的文件系统。

	-target：是存放框架 tar 包的目标路径，`#`后跟它的本地化别名(可选)。然后就会上传 tar 到指定的目录。不需要 gzip，因为 jar 文件是已经压缩了的。确保目标目录是所有用户都可读的。但是除了管理员之外，其他人不能写的，来保护集群安全性。

> -fs: The target file system. Defaults to the default filesystem set by fs.defaultFS.

> -target is the target location of the framework tarball, optionally followed by a # with the localized alias. It then uploads the tar to the specified directory. gzip is not needed since the jar files are already compressed. Make sure the target directory is readable by all users but it is not writable by others than administrators to protect cluster security.

> 2.Configure mapreduce.application.framework.path to point to the location where the archive is located. As when specifying distributed cache files for a job, this is a URL that also supports creating an alias for the archive if a URL fragment is specified. For example, hdfs:///mapred/framework/hadoop-mapreduce-3.2.1.tar.gz#mrframework will be localized as mrframework rather than hadoop-mapreduce-3.2.1.tar.gz.

- 2.**配置 `mapreduce.application.framework.path`，来指向 archive 的位置**。

	当为一个 job 指定了分布式缓存文件，那这就是一个支持创建 archive 别名的 URL，如果指定了一个 URL 片段。

	例如，`hdfs:///mapred/framework/hadoop-mapreduce-3.2.1.tar.gz#mrframework` 将被本地化为 `mrframework`，而不是 `hadoop-mapreduce-3.2.1.tar.gz`

> 3.Configure mapreduce.application.classpath to set the proper classpath to use with the MapReduce archive configured above. If the frameworkuploader tool is used, it uploads all dependencies and returns the value that needs to be configured here. NOTE: An error occurs if mapreduce.application.framework.path is configured but mapreduce.application.classpath does not reference the base name of the archive path or the alias if an alias was specified.

- 3.**配置 `mapreduce.application.classpath` ，来配置与上面配置的 MapReduce archive 一起使用的正确的 classpath**。

	如果使用 frameworkuploader 工具，它会上传所有依赖项，并返回这里需要配置的值。

	注意:如果配置了 `mapreduce.application.framework.path`，但 `mapreduce.application.classpath` 没有指向 archive 路径或别名，则会发生错误。

> Note that the location of the MapReduce archive can be critical to job submission and job startup performance. If the archive is not located on the cluster’s default filesystem then it will be copied to the job staging directory for each job and localized to each node where the job’s tasks run. This will slow down job submission and task startup performance.

注意：**MapReduce archive 的位置对 job 提交和 job 启动的性能至关重要**。

**如果 archive 文件不在集群的默认文件系统中，那么它将为每个 job 复制到这个 job 暂存目录，并本地化到运行 job 任务的每个节点。这将降低 job 提交和任务启动性能**。

> If the archive is located on the default filesystem then the job client will not upload the archive to the job staging directory for each job submission. However if the archive path is not readable by all cluster users then the archive will be localized separately for each user on each node where tasks execute. This can cause unnecessary duplication in the distributed cache.

**如果 archive 位于默认文件系统中，那么 job 客户端将不会为每个 job 提交，上传 archive 到 job 暂存目录中**。

但是，**如果 archive 路径不是所有集群用户都可读的，那么 archive 将对执行任务的每个节点上的每个用户分别进行本地化。这可能会导致分布式缓存中出现不必要的重复。**

> When working with a large cluster it can be important to increase the replication factor of the archive to increase its availability. This will spread the load when the nodes in the cluster localize the archive for the first time.

**在使用大型集群时，增加 archive 的副本因子，以提高其可用性是很重要的**。这将在集群节点第一次本地化 archive 时分散负载。

> The frameworkuploader tool mentioned above has additional parameters that help to adjust performance:

上面提到的 frameworkuploader 工具有额外的参数来帮助调整性能:

> -initialReplication: This is the replication count that the framework tarball is created with. It is safe to leave this value at the default 3. This is the tested scenario.

-initialreplication：这是创建框架 tar 包时使用的副本计数。保留默认值是 3 安全的。这是经过测试的场景。

> -finalReplication: The uploader tool sets the replication once all blocks are collected and uploaded. If quick initial startup is required, then it is advised to set this to the commissioned node count divided by two but not more than 512. This will leverage HDFS to spread the tarball in a distributed manner. Once the jobs start they will likely hit a local HDFS node to localize from or they can select from a wide set of additional source nodes. If this is is set to a low value like 10, then the output bandwidth of those replicated nodes will affect how fast the first job will run. The replication count can be manually reduced to a low value like 10 once all the jobs started in the cluster to save disk space.

-finalReplication：在所有的块被收集和上传后，上传工具设置副本。如果需要快速的初始启动，那么建议将其设置为委托节点数除以2，但不超过512。这将利用 HDFS 以分布式的方式分发 tar 包。一旦 jobs 启动，它们很可能会命中本地 HDFS 节点进行本地化，或者它们可以从大量其他源节点中进行选择。如果将这个值设置为较低的值，如10，那么副本节点的输出带宽将影响第一个 job 的运行速度。在集群中启动所有 job 后，可以手动将副本计数减少到10这样的低值，以节省磁盘空间。

> -acceptableReplication: The tool will wait until the tarball has been replicated this number of times before exiting. This should be a replication count less than or equal to the value in finalReplication. This is typically a 90% of the value in finalReplication to accomodate failing nodes.

-acceptableReplication：该工具将等待，直到 tar 包被复制这个次数，然后退出。这应该是一个小于或等于 `finalReplication`的副本计数。通常，这个值是`finalReplication` 的 90%，用于容纳失败节点。

> -timeout: A timeout in seconds to wait to reach acceptableReplication before the tool exits. The tool logs an error otherwise and returns.

-timeout：在工具退出之前，等待到达`acceptableReplication`的超时时间。否则，该工具会记录一个错误并返回。

## 4、MapReduce Archives and Classpath Configuration

> Setting a proper classpath for the MapReduce archive depends upon the composition of the archive and whether it has any additional dependencies. For example, the archive can contain not only the MapReduce jars but also the necessary YARN, HDFS, and Hadoop Common jars and all other dependencies. In that case, mapreduce.application.classpath would be configured to something like the following example, where the archive basename is hadoop-mapreduce-3.2.1.tar.gz and the archive is organized internally similar to the standard Hadoop distribution archive:

**为 MapReduce archive 设置的 classpath 取决于 archive 的组成，以及它是否有任何额外的依赖项**。

例如，**archive 文件不仅可以包含 MapReduce jar 文件，还可以包含必要的 YARN、HDFS 和 Hadoop Common jar 以及所有其他依赖项**。

在这种情况下，`mapreduce.application.classpath`应该配置为如下示例所示。

例如，archive basename 是 `hadoop-mapreduce-3.2.1.tar.gz`，archive 内部组织类似于标准的 Hadoop 发行版 archive :

	$HADOOP_CONF_DIR,$PWD/hadoop-mapreduce-3.2.1.tar.gz/hadoop-mapreduce-3.2.1/share/hadoop/mapreduce/*,$PWD/hadoop-mapreduce-3.2.1.tar.gz/hadoop-mapreduce-3.2.1/share/hadoop/mapreduce/lib/*,$PWD/hadoop-mapreduce-3.2.1.tar.gz/hadoop-mapreduce-3.2.1/share/hadoop/common/*,$PWD/hadoop-mapreduce-3.2.1.tar.gz/hadoop-mapreduce-3.2.1/share/hadoop/common/lib/*,$PWD/hadoop-mapreduce-3.2.1.tar.gz/hadoop-mapreduce-3.2.1/share/hadoop/yarn/*,$PWD/hadoop-mapreduce-3.2.1.tar.gz/hadoop-mapreduce-3.2.1/share/hadoop/yarn/lib/*,$PWD/hadoop-mapreduce-3.2.1.tar.gz/hadoop-mapreduce-3.2.1/share/hadoop/hdfs/*,$PWD/hadoop-mapreduce-3.2.1.tar.gz/hadoop-mapreduce-3.2.1/share/hadoop/hdfs/lib/*

> Another possible approach is to have the archive consist of just the MapReduce jars and have the remaining dependencies picked up from the Hadoop distribution installed on the nodes. In that case, the above example would change to something like the following:

另一种可能的方法是**让 archive 只包含 MapReduce jar，并在节点上安装从 Hadoop 发行版获得的剩余依赖项**。在这种情况下，上面的例子会变成如下内容:

	$HADOOP_CONF_DIR,$PWD/hadoop-mapreduce-3.2.1.tar.gz/hadoop-mapreduce-3.2.1/share/hadoop/mapreduce/*,$PWD/hadoop-mapreduce-3.2.1.tar.gz/hadoop-mapreduce-3.2.1/share/hadoop/mapreduce/lib/*,$HADOOP_COMMON_HOME/share/hadoop/common/*,$HADOOP_COMMON_HOME/share/hadoop/common/lib/*,$HADOOP_HDFS_HOME/share/hadoop/hdfs/*,$HADOOP_HDFS_HOME/share/hadoop/hdfs/lib/*,$HADOOP_YARN_HOME/share/hadoop/yarn/*,$HADOOP_YARN_HOME/share/hadoop/yarn/lib/*

> The frameworkuploader tool has the following arguments to control which jars end up in the framework tarball:

frameworkuploader 工具有以下参数，来控制哪些 jar 文件在框架 tar 包中结束:

> -input: This is the input classpath that is iterated through. jars files found will be added to the tarball. It defaults to the classpath as returned by the hadoop classpath command.

-input：这是输入 classpath 。找到的 jar 文件将被添加到 tar 包中。默认为 `hadoop classpath` 命令返回的 classpath。

> -blacklist: This is a comma separated regex array to filter the jar file names to exclude from the class path. It can be used for example to exclude test jars or Hadoop services that are not necessary to localize.

-blacklist：这是一个逗号分隔的正则数组，用于过滤 jar 文件名，将其排除在类路径之外。例如，可以使用它来排除不需要本地化的测试 jar 或 Hadoop 服务。

> -whitelist: This is a comma separated regex array to include certain jar files. This can be used to provide additional security, so that no external source can include malicious code in the classpath when the tool runs.

-whitelist：这是一个逗号分隔的正则数组，用于包含特定的 jar 文件。这可以用来提供额外的安全性，以便当工具运行时，任何外部源都不能在 classpath 中包含恶意代码。

> -nosymlink: This flag can be used to exclude symlinks that point to the same directory. This is not widely used. For example, /a/foo.jar and a symlink /a/bar.jar that points to /a/foo.jar would normally add foo.jar and bar.jar to the tarball as separate files despite them actually being the same file. This flag would make the tool exclude /a/bar.jar so only one copy of the file is added.

-nosymlink：这个标志可以用来排除指向同一目录的符号链接。这并没有被广泛使用。

例如，`/a/foo.jar`和指向`/a/foo.jar`的符号链接`/a/bar.jar`通常会将`foo.jar`和`bar.jar`作为单独的文件添加到 tar 包中，尽管它们实际上是相同的文件。这个标志将使工具排除`/a/bar.jar`，因此只添加文件的一个副本。

> If shuffle encryption is also enabled in the cluster, then we could meet the problem that MR job get failed with exception like below:

**如果集群中启用了 shuffle 加密，则我们会遇到 MR job 异常失败的问题**，如下所示：

	2014-10-10 02:17:16,600 WARN [fetcher#1] org.apache.hadoop.mapreduce.task.reduce.Fetcher: Failed to connect to junpingdu-centos5-3.cs1cloud.internal:13562 with 1 map outputs
	javax.net.ssl.SSLHandshakeException: sun.security.validator.ValidatorException: PKIX path building failed: sun.security.provider.certpath.SunCertPathBuilderException: unable to find valid certification path to requested target
	    at com.sun.net.ssl.internal.ssl.Alerts.getSSLException(Alerts.java:174)
	    at com.sun.net.ssl.internal.ssl.SSLSocketImpl.fatal(SSLSocketImpl.java:1731)
	    at com.sun.net.ssl.internal.ssl.Handshaker.fatalSE(Handshaker.java:241)
	    at com.sun.net.ssl.internal.ssl.Handshaker.fatalSE(Handshaker.java:235)
	    at com.sun.net.ssl.internal.ssl.ClientHandshaker.serverCertificate(ClientHandshaker.java:1206)
	    at com.sun.net.ssl.internal.ssl.ClientHandshaker.processMessage(ClientHandshaker.java:136)
	    at com.sun.net.ssl.internal.ssl.Handshaker.processLoop(Handshaker.java:593)
	    at com.sun.net.ssl.internal.ssl.Handshaker.process_record(Handshaker.java:529)
	    at com.sun.net.ssl.internal.ssl.SSLSocketImpl.readRecord(SSLSocketImpl.java:925)
	    at com.sun.net.ssl.internal.ssl.SSLSocketImpl.performInitialHandshake(SSLSocketImpl.java:1170)
	    at com.sun.net.ssl.internal.ssl.SSLSocketImpl.startHandshake(SSLSocketImpl.java:1197)
	    at com.sun.net.ssl.internal.ssl.SSLSocketImpl.startHandshake(SSLSocketImpl.java:1181)
	    at sun.net.www.protocol.https.HttpsClient.afterConnect(HttpsClient.java:434)
	    at sun.net.www.protocol.https.AbstractDelegateHttpsURLConnection.setNewClient(AbstractDelegateHttpsURLConnection.java:81)
	    at sun.net.www.protocol.https.AbstractDelegateHttpsURLConnection.setNewClient(AbstractDelegateHttpsURLConnection.java:61)
	    at sun.net.www.protocol.http.HttpURLConnection.writeRequests(HttpURLConnection.java:584)
	    at sun.net.www.protocol.http.HttpURLConnection.getInputStream(HttpURLConnection.java:1193)
	    at java.net.HttpURLConnection.getResponseCode(HttpURLConnection.java:379)
	    at sun.net.www.protocol.https.HttpsURLConnectionImpl.getResponseCode(HttpsURLConnectionImpl.java:318)
	    at org.apache.hadoop.mapreduce.task.reduce.Fetcher.verifyConnection(Fetcher.java:427)
	....

这是因为 MR 客户端(从 HDFS 部署)不能访问本地 F S中`$HADOOP_CONF_DIR`目录下的`ssl-client.xml`。

要解决这个问题，我们可以将带有`ssl-client.xml`的目录添加到`mapreduce.application.classpath`中指定的 MR 的类路径中。

如上所述。为了避免 MR 应用程序受到其他本地配置的影响，最好为放置`ssl-client.xml`创建一个专用目录。例如:`$HADOOP_CONF_DIR`下的子目录，如:`$HADOOP_CONF_DIR/security`。

> This is because MR client (deployed from HDFS) cannot access ssl-client.xml in local FS under directory of $HADOOP_CONF_DIR. To fix the problem, we can add the directory with ssl-client.xml to the classpath of MR which is specified in “mapreduce.application.classpath” as mentioned above. To avoid MR application being affected by other local configurations, it is better to create a dedicated directory for putting ssl-client.xml, e.g. a sub-directory under $HADOOP_CONF_DIR, like: $HADOOP_CONF_DIR/security.

> The framework upload tool can be use to collect cluster jars that the MapReduce AM, mappers and reducers will use. It returns logs that provide the suggested configuration values

框架上传工具可以用来收集 MapReduce AM 、mappers 和 reducers 将使用的集群 jar 文件。它返回提供建议配置值的日志：

	INFO uploader.FrameworkUploader: Uploaded hdfs://mynamenode/mapred/framework/mr-framework.tar#mr-framework

	INFO uploader.FrameworkUploader: Suggested mapreduce.application.classpath $PWD/mr-framework/*

设置`mapreduce.application.framework.path`指向上面第一个日志记录的值。

设置`mapreduce.application.classpath`指向上面第二个日志记录的值。

> Set mapreduce.application.framework.path to the first and mapreduce.application.classpath to the second logged value above respectively.