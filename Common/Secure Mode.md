# Secure Mode

[TOC]

## 1、Introduction

> This document describes how to configure authentication for Hadoop in secure mode. When Hadoop is configured to run in secure mode, each Hadoop service and each user must be authenticated by Kerberos.

本文档描述了 Hadoop 安全模式下配置身份验证的方法。

当 Hadoop 配置为安全模式运行时，每个 Hadoop 服务和每个用户都必须通过 Kerberos 进行身份验证。

> Forward and reverse host lookup for all service hosts must be configured correctly to allow services to authenticate with each other. Host lookups may be configured using either DNS or /etc/hosts files. Working knowledge of Kerberos and DNS is recommended before attempting to configure Hadoop services in Secure Mode.

必须正确配置对所有服务主机的正向和反向主机查找，以允许服务彼此进行身份验证。

主机查找可以使用 DNS 或 /etc/hosts 文件配置。

在使用安全模式配置 Hadoop 服务之前，建议先了解 Kerberos 和 DNS 的工作原理。

> Security features of Hadoop consist of [Authentication](https://hadoop.apache.org/docs/r3.2.1/hadoop-project-dist/hadoop-common/SecureMode.html#Authentication), [Service Level Authorization](https://hadoop.apache.org/docs/r3.2.1/hadoop-project-dist/hadoop-common/ServiceLevelAuth.html), [Authentication for Web Consoles](https://hadoop.apache.org/docs/r3.2.1/hadoop-project-dist/hadoop-common/HttpAuthentication.html) and [Data Confidentiality](https://hadoop.apache.org/docs/r3.2.1/hadoop-project-dist/hadoop-common/SecureMode.html#Data_confidentiality).

Hadoop 的安全特性包括身份验证、服务级别身份验证、Web控制台的身份验证和数据机密性。

## 2、Authentication

### 2.1、End User Accounts

> When service level authentication is turned on, end users must authenticate themselves before interacting with Hadoop services. The simplest way is for a user to authenticate interactively using the [Kerberos kinit command](http://web.mit.edu/kerberos/krb5-1.12/doc/user/user_commands/kinit.html). Programmatic authentication using Kerberos keytab files may be used when interactive login with kinit is infeasible.

当开启服务级别身份验证时，终端用户必须在与 Hadoop 服务交互之前对自己进行身份验证。

最简单的方法是让用户使用 Kerberos 的 kinit 命令进行交互身份验证。

当无法使用 kinit 进行交互登录时，可以使用 Kerberos keytab 文件进行编程身份验证。

### 2.2、User Accounts for Hadoop Daemons

> Ensure that HDFS and YARN daemons run as different Unix users, e.g. hdfs and yarn. Also, ensure that the MapReduce JobHistory server runs as different user such as mapred.

确保 HDFS 和 YARN 的守护进程以不同的 Unix 用户运行，例如 hdfs 和 yarn。

同时，确保 MapReduce JobHistory 服务以不同的用户(如mapred)运行。

> It’s recommended to have them share a Unix group, e.g. hadoop. See also [“Mapping from user to group”](https://hadoop.apache.org/docs/r3.2.1/hadoop-project-dist/hadoop-common/SecureMode.html#Mapping_from_user_to_group) for group management.

建议让它们共享一个 Unix 组，例如 hadoop。有关组管理，请参见 Mapping from user to group。

User:Group    |  Daemons
---|:---
hdfs:hadoop   |  NameNode, Secondary NameNode, JournalNode, DataNode
yarn:hadoop	  |  ResourceManager, NodeManager
mapred:hadoop |	 MapReduce JobHistory Server

### 2.3、Kerberos principals for Hadoop Daemons

> Each Hadoop Service instance must be configured with its Kerberos principal and keytab file location.

每个 Hadoop 服务实例都必须使用其 Kerberos 主体和 keytab 文件位置配置。

> The general format of a Service principal is ServiceName/_HOST@REALM.TLD. e.g. dn/_HOST@EXAMPLE.COM.

服务主体的一般格式是 `ServiceName/_HOST@REALM.TLD`。如，`dn/_HOST@EXAMPLE.COM` 。

> Hadoop simplifies the deployment of configuration files by allowing the hostname component of the service principal to be specified as the `_HOST` wildcard. Each service instance will substitute `_HOST` with its own fully qualified hostname at runtime. This allows administrators to deploy the same set of configuration files on all nodes. However, the keytab files will be different.

Hadoop允许将服务主体的主机名组件指定为 `_HOST` 通配符，从而简化了配置文件的部署。

每个服务实例将在运行时用它自己的完全限定主机名替换 `_HOST`。

这允许管理员在所有节点上部署相同的配置文件集。然而，keytab 文件将是不同的。

#### 2.3.1、HDFS

> The NameNode keytab file, on each NameNode host, should look like the following:

每个 NameNode 主机上的 NameNode keytab 文件应该如下所示：

```sh
$ klist -e -k -t /etc/security/keytab/nn.service.keytab
Keytab name: FILE:/etc/security/keytab/nn.service.keytab
KVNO Timestamp         Principal
   4 07/18/11 21:08:09 nn/full.qualified.domain.name@REALM.TLD (AES-256 CTS mode with 96-bit SHA-1 HMAC)
   4 07/18/11 21:08:09 nn/full.qualified.domain.name@REALM.TLD (AES-128 CTS mode with 96-bit SHA-1 HMAC)
   4 07/18/11 21:08:09 nn/full.qualified.domain.name@REALM.TLD (ArcFour with HMAC/md5)
   4 07/18/11 21:08:09 host/full.qualified.domain.name@REALM.TLD (AES-256 CTS mode with 96-bit SHA-1 HMAC)
   4 07/18/11 21:08:09 host/full.qualified.domain.name@REALM.TLD (AES-128 CTS mode with 96-bit SHA-1 HMAC)
   4 07/18/11 21:08:09 host/full.qualified.domain.name@REALM.TLD (ArcFour with HMAC/md5)
```

> The Secondary NameNode keytab file, on that host, should look like the following:

在那个主机上的 Secondary NameNode keytab 文件应该如下所示：

```sh
$ klist -e -k -t /etc/security/keytab/sn.service.keytab
Keytab name: FILE:/etc/security/keytab/sn.service.keytab
KVNO Timestamp         Principal
   4 07/18/11 21:08:09 sn/full.qualified.domain.name@REALM.TLD (AES-256 CTS mode with 96-bit SHA-1 HMAC)
   4 07/18/11 21:08:09 sn/full.qualified.domain.name@REALM.TLD (AES-128 CTS mode with 96-bit SHA-1 HMAC)
   4 07/18/11 21:08:09 sn/full.qualified.domain.name@REALM.TLD (ArcFour with HMAC/md5)
   4 07/18/11 21:08:09 host/full.qualified.domain.name@REALM.TLD (AES-256 CTS mode with 96-bit SHA-1 HMAC)
   4 07/18/11 21:08:09 host/full.qualified.domain.name@REALM.TLD (AES-128 CTS mode with 96-bit SHA-1 HMAC)
   4 07/18/11 21:08:09 host/full.qualified.domain.name@REALM.TLD (ArcFour with HMAC/md5)
```

> The DataNode keytab file, on each host, should look like the following:

在每个主机上的DataNode keytab 文件应该如下所示：

```sh
$ klist -e -k -t /etc/security/keytab/dn.service.keytab
Keytab name: FILE:/etc/security/keytab/dn.service.keytab
KVNO Timestamp         Principal
   4 07/18/11 21:08:09 dn/full.qualified.domain.name@REALM.TLD (AES-256 CTS mode with 96-bit SHA-1 HMAC)
   4 07/18/11 21:08:09 dn/full.qualified.domain.name@REALM.TLD (AES-128 CTS mode with 96-bit SHA-1 HMAC)
   4 07/18/11 21:08:09 dn/full.qualified.domain.name@REALM.TLD (ArcFour with HMAC/md5)
   4 07/18/11 21:08:09 host/full.qualified.domain.name@REALM.TLD (AES-256 CTS mode with 96-bit SHA-1 HMAC)
   4 07/18/11 21:08:09 host/full.qualified.domain.name@REALM.TLD (AES-128 CTS mode with 96-bit SHA-1 HMAC)
   4 07/18/11 21:08:09 host/full.qualified.domain.name@REALM.TLD (ArcFour with HMAC/md5)
```

#### 2.3.2、YARN

> The ResourceManager keytab file, on the ResourceManager host, should look like the following:

在 ResourceManager 主机上的 ResourceManager keytab 文件应该如下所示：

```sh
$ klist -e -k -t /etc/security/keytab/rm.service.keytab
Keytab name: FILE:/etc/security/keytab/rm.service.keytab
KVNO Timestamp         Principal
   4 07/18/11 21:08:09 rm/full.qualified.domain.name@REALM.TLD (AES-256 CTS mode with 96-bit SHA-1 HMAC)
   4 07/18/11 21:08:09 rm/full.qualified.domain.name@REALM.TLD (AES-128 CTS mode with 96-bit SHA-1 HMAC)
   4 07/18/11 21:08:09 rm/full.qualified.domain.name@REALM.TLD (ArcFour with HMAC/md5)
   4 07/18/11 21:08:09 host/full.qualified.domain.name@REALM.TLD (AES-256 CTS mode with 96-bit SHA-1 HMAC)
   4 07/18/11 21:08:09 host/full.qualified.domain.name@REALM.TLD (AES-128 CTS mode with 96-bit SHA-1 HMAC)
   4 07/18/11 21:08:09 host/full.qualified.domain.name@REALM.TLD (ArcFour with HMAC/md5)
```

> The NodeManager keytab file, on each host, should look like the following:

在每个主机上的 NodeManager keytab 文件应该如下所示：

```sh
$ klist -e -k -t /etc/security/keytab/nm.service.keytab
Keytab name: FILE:/etc/security/keytab/nm.service.keytab
KVNO Timestamp         Principal
   4 07/18/11 21:08:09 nm/full.qualified.domain.name@REALM.TLD (AES-256 CTS mode with 96-bit SHA-1 HMAC)
   4 07/18/11 21:08:09 nm/full.qualified.domain.name@REALM.TLD (AES-128 CTS mode with 96-bit SHA-1 HMAC)
   4 07/18/11 21:08:09 nm/full.qualified.domain.name@REALM.TLD (ArcFour with HMAC/md5)
   4 07/18/11 21:08:09 host/full.qualified.domain.name@REALM.TLD (AES-256 CTS mode with 96-bit SHA-1 HMAC)
   4 07/18/11 21:08:09 host/full.qualified.domain.name@REALM.TLD (AES-128 CTS mode with 96-bit SHA-1 HMAC)
   4 07/18/11 21:08:09 host/full.qualified.domain.name@REALM.TLD (ArcFour with HMAC/md5)
```

#### 2.3.3、MapReduce JobHistory Server

> The MapReduce JobHistory Server keytab file, on that host, should look like the following:

在那个主机上的 MapReduce JobHistory Server keytab 文件应该如下所示：

```sh
$ klist -e -k -t /etc/security/keytab/jhs.service.keytab
Keytab name: FILE:/etc/security/keytab/jhs.service.keytab
KVNO Timestamp         Principal
   4 07/18/11 21:08:09 jhs/full.qualified.domain.name@REALM.TLD (AES-256 CTS mode with 96-bit SHA-1 HMAC)
   4 07/18/11 21:08:09 jhs/full.qualified.domain.name@REALM.TLD (AES-128 CTS mode with 96-bit SHA-1 HMAC)
   4 07/18/11 21:08:09 jhs/full.qualified.domain.name@REALM.TLD (ArcFour with HMAC/md5)
   4 07/18/11 21:08:09 host/full.qualified.domain.name@REALM.TLD (AES-256 CTS mode with 96-bit SHA-1 HMAC)
   4 07/18/11 21:08:09 host/full.qualified.domain.name@REALM.TLD (AES-128 CTS mode with 96-bit SHA-1 HMAC)
   4 07/18/11 21:08:09 host/full.qualified.domain.name@REALM.TLD (ArcFour with HMAC/md5)
```

### 2.4、Mapping from Kerberos principals to OS user accounts

> Hadoop maps Kerberos principals to OS user (system) accounts using rules specified by hadoop.security.auth_to_local. How Hadoop evaluates these rules is determined by the setting of hadoop.security.auth_to_local.mechanism.

Hadoop 使用 `hadoop.security.auth_to_local` 指定的规则将 Kerberos 主体映射到操作系统用户帐户。

Hadoop 如何评估这些规则取决于 `hadoop.security.auth_to_local.mechanism` 的设置。

> In the default hadoop mode a Kerberos principal must be matched against a rule that transforms the principal to a simple form, i.e. a user account name without ‘@’ or ‘/’, otherwise a principal will not be authorized and a error will be logged. In case of the MIT mode the rules work in the same way as the auth_to_local in [Kerberos configuration file (krb5.conf)](http://web.mit.edu/Kerberos/krb5-latest/doc/admin/conf_files/krb5_conf.html) and the restrictions of hadoop mode do not apply. If you use MIT mode it is suggested to use the same auth_to_local rules that are specified in your /etc/krb5.conf as part of your default realm and keep them in sync. In both hadoop and MIT mode the rules are being applied (with the exception of DEFAULT) to all principals regardless of their specified realm. Also, note you should not rely on the auth_to_local rules as an ACL and use proper (OS) mechanisms.

在默认的 hadoop 模式中，Kerberos 主体必须与将主体转换为简单形式的规则相匹配，即不带 `@` 或 `/` 的用户帐户名，否则主体将得不到授权，并会记录错误。

在 MIT 模式下，规则的工作方式与 Kerberos 配置文件(krb5.conf)中的 auth_to_local 相同，hadoop 模式的限制不适用。

如果使用 MIT 模式，建议使用 `/etc/krb5.conf` 中指定的相同的 auth_to_local 规则作为默认域的一部分，并使它们保持同步。

在 hadoop 和 MIT 模式中，规则都被应用到所有主体(DEFAULT例外)，而不考虑其指定的领域。

另外，请注意，作为 ACL，你不应该依赖于 auth_to_local 规则，并使用适当的(OS)机制。

> Possible values for auth_to_local are:

auth_to_local 可能的值是:

> RULE:exp The local name will be formulated from exp. The format for exp is [n:string](regexp)s/pattern/replacement/g. The integer n indicates how many components the target principal should have. If this matches, then a string will be formed from string, substituting the realm of the principal for $0 and the n’th component of the principal for $n (e.g., if the principal was johndoe/admin then [2:$2$1foo] would result in the string adminjohndoefoo). If this string matches regexp, then the s//[g] substitution command will be run over the string. The optional g will cause the substitution to be global over the string, instead of replacing only the first match in the string. As an extension to MIT, Hadoop auth_to_local mapping supports the /L flag that lowercases the returned name.

- RULE:exp

本地名字将从 exp 指定。

exp 的格式是 `[n:string](regexp)s/pattern/replacement/g`。

整数 n 表示目标主体应该有多少个组件。如果这个匹配，那么 string 将由字符串组成，将主体的域替换为 $0，将主体的第 n 个组件替换为 $n

(例如，如果主体是 johndoe/admin，那么 `[2:$2$1foo]`将得到字符串 `adminjohndoefoo`)。

如果该字符串匹配 regexp，则将在该字符串上运行 `s//[g]` 替换命令。

可选的 g 将导致替换在整个字符串的全局替换，而不是只替换字符串中的第一个匹配项。

作为对 MIT 的扩展，Hadoop auth_to_local 映射支持将返回的名称小写的 /L 标志。

> DEFAULT Picks the first component of the principal name as the system user name if and only if the realm matches the default_realm (usually defined in /etc/krb5.conf). e.g. The default rule maps the principal host/full.qualified.domain.name@MYREALM.TLD to system user host if the default realm is MYREALM.TLD.

- DEFAULT

选择主体名称的第一个组件作为系统用户名，当且仅当领域匹配 default_realm(通常在 `/etc/krb5.conf` 中定义)时。

例如，如果默认域是 `MYREALM.TLD`，则默认规则将主体 `host/full.qualified.domain.name@MYREALM.TLD` 映射到系统用户主机。

> In case no rules are specified Hadoop defaults to using DEFAULT, which is probably not suitable to most of the clusters.

如果没有指定规则，Hadoop 默认使用 DEFAULT，这可能不适用于大多数集群。

> Please note that Hadoop does not support multiple default realms (e.g like Heimdal does). Also, Hadoop does not do a verification on mapping whether a local system account exists.

请注意，Hadoop 不支持多个默认域(比如 Heimdal 就支持)。而且，Hadoop 不会对映射是否存在于本地系统帐户进行验证。

### 2.5、Example rules

> In a typical cluster HDFS and YARN services will be launched as the system hdfs and yarn users respectively. hadoop.security.auth_to_local can be configured as follows:

在一个典型的集群中，HDFS 和 YARN 服务将分别作为系统 hdfs 和 yarn 用户启动。

`hadoop.security.auth_to_local` 可以按如下配置:

```xml
<property>
  <name>hadoop.security.auth_to_local</name>
  <value>
    RULE:[2:$1/$2@$0]([ndj]n/.*@REALM.\TLD)s/.*/hdfs/
    RULE:[2:$1/$2@$0]([rn]m/.*@REALM\.TLD)s/.*/yarn/
    RULE:[2:$1/$2@$0](jhs/.*@REALM\.TLD)s/.*/mapred/
    DEFAULT
  </value>
</property>
```

> This would map any principal nn, dn, jn on any host from realm REALM.TLD to the local system account hdfs. Secondly it would map any principal rm, nm on any host from REALM.TLD to the local system account yarn. Thirdly, it would map the principal jhs on any host from realm REALM.TLD to the local system account mapred. Finally, any principal on any host from the default realm will be mapped to the user component of that principal.

这将映射来自 `REALM.TLD` 的任何主机上的任何主体 nn、dn、jn。

其次，它将映射来自 `REALM.TLD` 的任何主机上的任何主体 rm、nm。

第三，它将映射来自 `REALM.TLD` 的任何主机上的主体 jhs 到本地系统帐户 mapred。

最后，来自默认域的任何主机上的任何主体都将被映射到该主体的用户组件。

> Custom rules can be tested using the hadoop kerbname command. This command allows one to specify a principal and apply Hadoop’s current auth_to_local ruleset.

可以使用 hadoop kerbname 命令测试自定义规则。这个命令允许指定一个主体，并应用 Hadoop 当前的 auth_to_local 规则集。

### 2.6、Mapping from user to group

> The system user to system group mapping mechanism can be configured via hadoop.security.group.mapping. See [Hadoop Groups Mapping](https://hadoop.apache.org/docs/r3.2.1/hadoop-project-dist/hadoop-common/GroupsMapping.html) for details.

系统用户到系统组的映射机制可以通过 `hadoop.security.group.mapping` 进行配置。

详细信息请参见 Hadoop Groups Mapping。

> Practically you need to manage SSO environment using Kerberos with LDAP for Hadoop in secure mode.

实际上，你需要在安全模式下使用 Kerberos 和用于 Hadoop 的 LDAP 来管理 SSO 环境。

### 2.7、Proxy user

> Some products such as Apache Oozie which access the services of Hadoop on behalf of end users need to be able to impersonate end users. See [the doc of proxy user](https://hadoop.apache.org/docs/r3.2.1/hadoop-project-dist/hadoop-common/Superusers.html) for details.

一些产品，如 Apache Oozie，它代表终端用户访问 Hadoop 服务，需要能够模拟终端用户。具体请参见 the doc of proxy user。

### 2.8、Secure DataNode

> Because the DataNode data transfer protocol does not use the Hadoop RPC framework, DataNodes must authenticate themselves using privileged ports which are specified by dfs.datanode.address and dfs.datanode.http.address. This authentication is based on the assumption that the attacker won’t be able to get root privileges on DataNode hosts.

由于 DataNode 数据传输协议不使用 Hadoop RPC 框架，因此 DataNode 必须使用 `dfs.datanode.address` 和 `dfs.datanode.http.address` 指定的特权端口进行身份验证。

这种身份验证基于这样的假设：攻击者无法获得 DataNode 主机上的根特权。

> When you execute the hdfs datanode command as root, the server process binds privileged ports at first, then drops privilege and runs as the user account specified by HDFS_DATANODE_SECURE_USER. This startup process uses the [jsvc program](https://commons.apache.org/proper/commons-daemon/jsvc.html) installed to JSVC_HOME. You must specify HDFS_DATANODE_SECURE_USER and JSVC_HOME as environment variables on start up (in hadoop-env.sh).

当以 root 身份执行 hdfs datanode 命令时，服务进程首先绑定特权端口，然后放弃特权，以 `HDFS_DATANODE_SECURE_USER` 指定的用户帐号运行。

这个启动过程使用安装在 JSVC_HOME 中的 jsvc 程序。必须在启动时指定 `HDFS_DATANODE_SECURE_USER` 和 `JSVC_HOME` 作为环境变量(在 hadoop-env.sh 中)。

> As of version 2.6.0, SASL can be used to authenticate the data transfer protocol. In this configuration, it is no longer required for secured clusters to start the DataNode as root using jsvc and bind to privileged ports. To enable SASL on data transfer protocol, set dfs.data.transfer.protection in hdfs-site.xml. A SASL enabled DataNode can be started in secure mode in following two ways: 1. Set a non-privileged port for dfs.datanode.address. 1. Set dfs.http.policy to HTTPS_ONLY or set dfs.datanode.http.address to a privileged port and make sure the HDFS_DATANODE_SECURE_USER and JSVC_HOME environment variables are specified properly as environment variables on start up (in hadoop-env.sh).

从版本 2.6.0 开始，可以使用 SASL 对数据传输协议进行身份验证。

在这个配置中，安全集群不再需要使用 jsvc 以 root 身份启动 DataNode ，并绑定到特权端口。

要在数据传输协议上启用 SASL，请在 hdfs-site.xml 上设置 `dfs.data.transfer.protection`。

启用了 SASL 的 DataNode 可以通过以下两种方式在安全模式下启动：

1. 为 `dfs.datanode.address` 设置一个非特权端口。

2. 设置 `dfs.http.policy` 为 `HTTPS_ONLY` 或设置 `dfs.datanode.http.address` 为一个特权端口，并确保 `HDFS_DATANODE_SECURE_USER` 和 `JSVC_HOME` 环境变量在启动时被正确地指定为环境变量(在 hadoop-env.sh 中)。

> In order to migrate an existing cluster that used root authentication to start using SASL instead, first ensure that version 2.6.0 or later has been deployed to all cluster nodes as well as any external applications that need to connect to the cluster. Only versions 2.6.0 and later of the HDFS client can connect to a DataNode that uses SASL for authentication of data transfer protocol, so it is vital that all callers have the correct version before migrating. After version 2.6.0 or later has been deployed everywhere, update configuration of any external applications to enable SASL. If an HDFS client is enabled for SASL, then it can connect successfully to a DataNode running with either root authentication or SASL authentication. Changing configuration for all clients guarantees that subsequent configuration changes on DataNodes will not disrupt the applications. Finally, each individual DataNode can be migrated by changing its configuration and restarting. It is acceptable to have a mix of some DataNodes running with root authentication and some DataNodes running with SASL authentication temporarily during this migration period, because an HDFS client enabled for SASL can connect to both.

为了迁移使用 root 身份验证而改用 SASL 启动的现有集群，首先要确保 2.6.0 或更高版本已经部署到所有集群节点以及任何需要连接到集群的外部应用程序。

只有版本 2.6.0 及更高版本的 HDFS 客户端可以连接到使用 SASL 进行数据传输协议认证的 DataNode，所以在迁移之前，所有调用者都必须有正确的版本。

在 2.6.0 或更高版本部署到各处之后，更新任何外部应用程序的配置以启用 SASL。

如果为 SASL 启用了 HDFS 客户端，那么它可以成功连接到运行 root 认证或 SASL 认证的 DataNode。

更改所有客户端的配置可以保证后续在 datanode 上的配置更改不会中断应用程序。

最后，可以通过更改每个 DataNode 的配置并重新启动来迁移它。

在迁移期间，一些 datanode 暂时运行 root 身份验证，一些 datanode 运行 SASL 身份验证，这是可以接受的，因为启用 SASL 的 HDFS 客户端可以同时连接到这两个 datanode。

## 3、Data confidentiality

### 3.1、Data Encryption on RPC

> The data transfered between hadoop services and clients can be encrypted on the wire. Setting hadoop.rpc.protection to privacy in core-site.xml activates data encryption.

hadoop 服务和客户端之间传输的数据可以在网络上加密。

在 core-site.xml 中，设置 `hadoop.rpc.protection` 为 `privacy` 激活数据加密。

### 3.2、Data Encryption on Block data transfer.

> You need to set dfs.encrypt.data.transfer to true in the hdfs-site.xml in order to activate data encryption for data transfer protocol of DataNode.

需要在 hdfs-site.xml 中设置 `dfs.encrypt.data.transfer` 为 true，以激活 DataNode 的数据传输协议的数据加密。

> Optionally, you may set dfs.encrypt.data.transfer.algorithm to either 3des or rc4 to choose the specific encryption algorithm. If unspecified, then the configured JCE default on the system is used, which is usually 3DES.

也可以设置 `dfs.encrypt.data.transfer.algorithm` 为 3des 或 rc4 来选择特定的加密算法。

如果未指定，则使用系统上配置的 JCE 默认值，通常是 3DES。

> Setting dfs.encrypt.data.transfer.cipher.suites to AES/CTR/NoPadding activates AES encryption. By default, this is unspecified, so AES is not used. When AES is used, the algorithm specified in dfs.encrypt.data.transfer.algorithm is still used during an initial key exchange. The AES key bit length can be configured by setting dfs.encrypt.data.transfer.cipher.key.bitlength to 128, 192 or 256. The default is 128.

设置 `dfs.encrypt.data.transfer.cipher.suites` 为 `AES/CTR/NoPadding` 激活 AES 加密。

默认情况下，这是未指定的，因此不使用 AES。

当使用 AES 时，使用 `dfs.encrypt.data.transfer.algorithm` 中指定的算法在初始密钥交换过程中仍然使用。

AES key 的位长度可以通过设置 `dfs.encrypt.data.transfer.cipher.key.bitlength` 为 128、192 或 256 来配置。默认值是128。

> AES offers the greatest cryptographic strength and the best performance. At this time, 3DES and RC4 have been used more often in Hadoop clusters.

AES 提供了最大的加密强度和最好的性能。现在，3DES 和 RC4 在 Hadoop 集群中使用得更频繁。

### 3.3、Data Encryption on HTTP

> Data transfer between Web-console and clients are protected by using SSL(HTTPS). SSL configuration is recommended but not required to configure Hadoop security with Kerberos.

web 控制台和客户端之间的数据传输使用 SSL(HTTPS)进行保护。

使用 Kerberos 配置 Hadoop 安全时，建议 SSL 配置，但不是必需的。

> To enable SSL for web console of HDFS daemons, set dfs.http.policy to either HTTPS_ONLY or HTTP_AND_HTTPS in hdfs-site.xml. Note KMS and HttpFS do not respect this parameter. See [Hadoop KMS](https://hadoop.apache.org/docs/r3.2.1/hadoop-kms/index.html) and [Hadoop HDFS over HTTP - Server Setup](https://hadoop.apache.org/docs/r3.2.1/hadoop-hdfs-httpfs/ServerSetup.html) for instructions on enabling KMS over HTTPS and HttpFS over HTTPS, respectively.

HDFS 守护进程的 web 控制台启用 SSL，需要在 hdfs-site.xml 中设置 `dfs.http.policy` 为 `HTTPS_ONLY` 或 `HTTP_AND_HTTPS`。

KMS 和 HttpFS 不属于该参数。。。。

> To enable SSL for web console of YARN daemons, set yarn.http.policy to HTTPS_ONLY in yarn-site.xml.

要在 YARN 进程的 web 控制台启用 SSL，可以在 yarn-site.xml 中设置 `yarn.http.policy` 为 `HTTPS_ONLY`。

> To enable SSL for web console of MapReduce JobHistory server, set mapreduce.jobhistory.http.policy to HTTPS_ONLY in mapred-site.xml.

如果需要在 MapReduce JobHistory server 的 web 控制台启用 SSL，请在 mapred-site.xml 中设置 `mapreduce.jobhistory.http.policy` 为 `HTTPS_ONLY` 。

## 4、Configuration

### 4.1、Permissions for both HDFS and local fileSystem paths

> The following table lists various paths on HDFS and local filesystems (on all nodes) and recommended permissions:

下面的表列出了 HDFS 和本地文件系统的不同路径和推荐的权限：

Filesystem | Path                  | User:Group  |  Permissions
---        |:---                   |:---         |:---
local | dfs.namenode.name.dir      | hdfs:hadoop |  drwx------
local | dfs.datanode.data.dir      | hdfs:hadoop |  drwx------
local | $HADOOP_LOG_DIR            | hdfs:hadoop |  drwxrwxr-x
local | $YARN_LOG_DIR              | yarn:hadoop |  drwxrwxr-x
local | yarn.nodemanager.local-dirs| yarn:hadoop |  drwxr-xr-x
local | yarn.nodemanager.log-dirs  | yarn:hadoop |  drwxr-xr-x
local | container-executor         | root:hadoop |  --Sr-s--*
local | conf/container-executor.cfg| root:hadoop |  r-------*
hdfs  | /                          | hdfs:hadoop |  drwxr-xr-x
hdfs  | /tmp                       | hdfs:hadoop |  drwxrwxrwxt
hdfs  | /user                      | hdfs:hadoop |  drwxr-xr-x
hdfs  | yarn.nodemanager.remote-app-log-dir        | yarn:hadoop   | drwxrwxrwxt
hdfs  | mapreduce.jobhistory.intermediate-done-dir | mapred:hadoop |drwxrwxrwxt
hdfs  | mapreduce.jobhistory.done-dir              | mapred:hadoop |drwxr-x---

### 4.2、Common Configurations

> In order to turn on RPC authentication in hadoop, set the value of hadoop.security.authentication property to "kerberos", and set security related settings listed below appropriately.

为了在 hadoop 中开启 RPC 身份验证，需要设置 `hadoop.security.authentication` 为 kerberos，并适当设置下面列出的与安全相关的设置。

> The following properties should be in the core-site.xml of all the nodes in the cluster.

下面的属性应该在集群中所有节点的 core-site.xml 中。

Parameter                      |  Value       |  Notes
---                            |:---          |:---
`hadoop.security.authentication` | kerberos     |  `simple` : No authentication. (default)  `kerberos` : Enable authentication by Kerberos.
`hadoop.security.authorization`  |  true        |  Enable [RPC service-level authorization](https://hadoop.apache.org/docs/r3.2.1/hadoop-project-dist/hadoop-common/ServiceLevelAuth.html).
`hadoop.rpc.protection`          |authentication|  `authentication` : authentication only (default); `integrity` : integrity check in addition to authentication; `privacy` : data encryption in addition to integrity
hadoop.security.auth_to_local  |RULE:exp1 RULE:exp2 … DEFAULT  | The value is string containing new line characters. See [Kerberos documentation](http://web.mit.edu/Kerberos/krb5-latest/doc/admin/conf_files/krb5_conf.html) for the format of exp.【包含换行符的字符串形式的值。】
`hadoop.proxyuser.superuser.hosts`|             |  comma separated hosts from which superuser access are allowed to impersonation. `*` means wildcard.【逗号分隔的主机，允许超级用户访问这些主机，进行模拟。`*`表示通配符】
`hadoop.proxyuser.superuser.groups`|            |  comma separated groups to which users impersonated by superuser belong. `*` means wildcard.【由超级用户模拟的用户所属的逗号分隔的组。。`*`表示通配符】

### 4.3、NameNode

Parameter                      |  Value       |  Notes
---                            |:---          |:---
`dfs.block.access.token.enable`  |  true        |  Enable HDFS block access tokens for secure operations.
`dfs.namenode.kerberos.principal`| `nn/_HOST@REALM.TLD` | Kerberos principal name for the NameNode.
`dfs.namenode.keytab.file`       | `/etc/security/keytab/nn.service.keytab` | Kerberos keytab file for the NameNode.
`dfs.namenode.kerberos.internal.spnego.principal`  |  `HTTP/_HOST@REALM.TLD` | The server principal used by the NameNode for web UI SPNEGO authentication. The SPNEGO server principal begins with the prefix `HTTP/` by convention. If the value is '*', the web server will attempt to login with every principal specified in the keytab file `dfs.web.authentication.kerberos.keytab`. For most deployments this can be set to `${dfs.web.authentication.kerberos.principal}` i.e use the value of `dfs.web.authentication.kerberos.principal`.
`dfs.web.authentication.kerberos.keytab`  | `/etc/security/keytab/spnego.service.keytab`  |  SPNEGO keytab file for the NameNode. In HA clusters this setting is shared with the Journal Nodes.

> The following settings allow configuring SSL access to the NameNode web UI (optional).

通过以下配置，可以配置通过 SSL 访问 NameNode web UI(可选)。

Parameter                   |  Value       |  Notes
---                         |:---          |:---
`dfs.http.policy`             | `HTTP_ONLY` or `HTTPS_ONLY` or `HTTP_AND_HTTPS`  | `HTTPS_ONLY` turns off http access. This option takes precedence over the deprecated configuration `dfs.https.enable` and `hadoop.ssl.enabled`. If using SASL to authenticate data transfer protocol instead of running DataNode as root and using privileged ports, then this property must be set to `HTTPS_ONLY` to guarantee authentication of HTTP servers. (See `dfs.data.transfer.protection`.)
`dfs.namenode.https-address`  | 0.0.0.0:9871  |  This parameter is used in non-HA mode and without federation. See [HDFS High Availability](https://hadoop.apache.org/docs/r3.2.1/hadoop-project-dist/hadoop-hdfs/HDFSHighAvailabilityWithNFS.html#Deployment) and [HDFS Federation](https://hadoop.apache.org/docs/r3.2.1/hadoop-project-dist/hadoop-hdfs/Federation.html#Federation_Configuration) for details.
`dfs.https.enable`            |     true      |  This value is deprecated. Use `dfs.http.policy`

### 4.4、Secondary NameNode

Parameter                   |  Value       |  Notes
---                         |:---          |:---
`dfs.namenode.secondary.http-address` | 0.0.0.0:9868 | HTTP web UI address for the Secondary NameNode.
`dfs.namenode.secondary.https-address`|0.0.0.0:9869  | HTTPS web UI address for the Secondary NameNode.
`dfs.secondary.namenode.keytab.file`  | `/etc/security/keytab/sn.service.keytab`  | Kerberos keytab file for the Secondary NameNode.
`dfs.secondary.namenode.kerberos.principal` | `sn/_HOST@REALM.TLD` |   Kerberos principal name for the Secondary NameNode.
`dfs.secondary.namenode.kerberos.internal.spnego.principal` | `HTTP/_HOST@REALM.TLD`  |  The server principal used by the Secondary NameNode for web UI SPNEGO authentication. The SPNEGO server principal begins with the prefix `HTTP/` by convention. If the value is '*', the web server will attempt to login with every principal specified in the keytab file `dfs.web.authentication.kerberos.keytab`. For most deployments this can be set to `${dfs.web.authentication.kerberos.principal}` i.e use the value of `dfs.web.authentication.kerberos.principal`.

### 4.5、JournalNode

Parameter                   |  Value       |  Notes
---                         |:---          |:---
`dfs.journalnode.kerberos.principal` | `jn/_HOST@REALM.TLD`  |  Kerberos principal name for the JournalNode.
`dfs.journalnode.keytab.file` | `/etc/security/keytab/jn.service.keytab` | Kerberos keytab file for the JournalNode.
`dfs.journalnode.kerberos.internal.spnego.principal` | `HTTP/_HOST@REALM.TLD` | The server principal used by the JournalNode for web UI SPNEGO authentication when Kerberos security is enabled. The SPNEGO server principal begins with the prefix `HTTP/` by convention. If the value is '*', the web server will attempt to login with every principal specified in the keytab file `dfs.web.authentication.kerberos.keytab`. For most deployments this can be set to `${dfs.web.authentication.kerberos.principal}` i.e use the value of `dfs.web.authentication.kerberos.principal`.
`dfs.web.authentication.kerberos.keytab` | `/etc/security/keytab/spnego.service.keytab` |  SPNEGO keytab file for the JournalNode. In HA clusters this setting is shared with the Name Nodes.
`dfs.journalnode.https-address` | 0.0.0.0:8481  | HTTPS web UI address for the JournalNode.

### 4.6、DataNode

Parameter                   |  Value       |  Notes
---                         |:---          |:---
`dfs.datanode.data.dir.perm`  | 700        |
`dfs.datanode.address`  | 0.0.0.0:1004   | Secure DataNode must use privileged port in order to assure that the server was started securely. This means that the server must be started via jsvc. Alternatively, this must be set to a non-privileged port if using SASL to authenticate data transfer protocol. (See `dfs.data.transfer.protection`.)
`dfs.datanode.http.address`  |  0.0.0.0:1006 | Secure DataNode must use privileged port in order to assure that the server was started securely. This means that the server must be started via jsvc.
`dfs.datanode.https.address` | 0.0.0.0:9865  | HTTPS web UI address for the Data Node.
`dfs.datanode.kerberos.principal` | `dn/_HOST@REALM.TLD` |  Kerberos principal name for the DataNode.
`dfs.datanode.keytab.file` | `/etc/security/keytab/dn.service.keytab` |Kerberos keytab file for the DataNode.
`dfs.encrypt.data.transfer` | false | set to true when using data encryption
`dfs.encrypt.data.transfer.algorithm` |  | optionally set to 3des or rc4 when using data encryption to control encryption algorithm
`dfs.encrypt.data.transfer.cipher.suites`  |  | optionally set to `AES/CTR/NoPadding` to activate AES encryption when using data encryption
`dfs.encrypt.data.transfer.cipher.key.bitlength` |  | optionally set to 128, 192 or 256 to control key bit length when using AES with data encryption
`dfs.data.transfer.protection` |  | `authentication `: authentication only; `integrity` : integrity check in addition to authentication; `privacy` : data encryption in addition to integrity This property is unspecified by default. Setting this property enables SASL for authentication of data transfer protocol. If this is enabled, then `dfs.datanode.address` must use a non-privileged port, `dfs.http.policy` must be set to `HTTPS_ONLY` and the `HDFS_DATANODE_SECURE_USER` environment variable must be undefined when starting the DataNode process.

### 4.7、WebHDFS

Parameter                   |  Value       |  Notes
---                         |:---          |:---
`dfs.web.authentication.kerberos.principal` | `http/_HOST@REALM.TLD`  |Kerberos principal name for the WebHDFS. In HA clusters this setting is commonly used by the JournalNodes for securing access to the JournalNode HTTP server with SPNEGO.
`dfs.web.authentication.kerberos.keytab` | `/etc/security/keytab/http.service.keytab` | Kerberos keytab file for WebHDFS. In HA clusters this setting is commonly used the JournalNodes for securing access to the JournalNode HTTP server with SPNEGO.

### 4.8、ResourceManager

Parameter                   |  Value       |  Notes
---                         |:---          |:---
`yarn.resourcemanager.principal` | `rm/_HOST@REALM.TLD` | Kerberos principal name for the ResourceManager.
`yarn.resourcemanager.keytab` | `/etc/security/keytab/rm.service.keytab`  | Kerberos keytab file for the ResourceManager.
`yarn.resourcemanager.webapp.https.address`  | `${yarn.resourcemanager.hostname}:8090` | The https adddress of the RM web application for non-HA. In HA clusters, use `yarn.resourcemanager.webapp.https.address.rm-id` for each ResourceManager. See [ResourceManager High Availability](https://hadoop.apache.org/docs/r3.2.1/hadoop-yarn/hadoop-yarn-site/ResourceManagerHA.html#Configurations) for details.

### 4.9、NodeManager

Parameter                   |  Value       |  Notes
---                         |:---          |:---
`yarn.nodemanager.principal` | `nm/_HOST@REALM.TLD` |  Kerberos principal name for the NodeManager.
`yarn.nodemanager.keytab`  | `/etc/security/keytab/nm.service.keytab`  | Kerberos keytab file for the NodeManager.
`yarn.nodemanager.container-executor.class` | `org.apache.hadoop.yarn.server.nodemanager.LinuxContainerExecutor` | Use LinuxContainerExecutor.
`yarn.nodemanager.linux-container-executor.group` | hadoop | Unix group of the NodeManager.
`yarn.nodemanager.linux-container-executor.path` | `/path/to/bin/container-executor` | The path to the executable of Linux container executor.
`yarn.nodemanager.webapp.https.address` | 0.0.0.0:8044 | The https adddress of the NM web application.

### 4.10、Configuration for WebAppProxy

> The WebAppProxy provides a proxy between the web applications exported by an application and an end user. If security is enabled it will warn users before accessing a potentially unsafe web application. Authentication and authorization using the proxy is handled just like any other privileged web application.

WebAppProxy 提供了应用程序导出的 web 应用程序和终端用户之间的代理。

如果启用了安全模式，它将在访问可能不安全的 web 应用程序之前警告用户。使用代理进行身份验证和授权的处理与任何其他有特权的 web 应用程序一样。

Parameter                   |  Value       |  Notes
---                         |:---          |:---
`yarn.web-proxy.address`  | WebAppProxy host:port for proxy to AM web apps. | host:port if this is the same as `yarn.resourcemanager.webapp.address` or it is not defined then the ResourceManager will run the proxy otherwise a standalone proxy server will need to be launched.
`yarn.web-proxy.keytab` | `/etc/security/keytab/web-app.service.keytab` | Kerberos keytab file for the WebAppProxy.
`yarn.web-proxy.principal` | `wap/_HOST@REALM.TLD` | Kerberos principal name for the WebAppProxy.

### 4.11、LinuxContainerExecutor

> A ContainerExecutor used by YARN framework which define how any container launched and controlled.

一个由 YARN 框架使用的 ContainerExecutor，它定义了如何启动和控制任何容器。

> The following are the available in Hadoop YARN:

以下是 Hadoop YARN 提供的功能:

ContainerExecutor         |   Description
---|:---
DefaultContainerExecutor  |  The default executor which YARN uses to manage container execution. The container process has the same Unix user as the NodeManager.【YARN用来管理容器执行的默认执行器。容器进程与NodeManager是相同的Unix用户。】
LinuxContainerExecutor    |  Supported only on GNU/Linux, this executor runs the containers as either the YARN user who submitted the application (when full security is enabled) or as a dedicated user (defaults to nobody) when full security is not enabled. When full security is enabled, this executor requires all user accounts to be created on the cluster nodes where the containers are launched. It uses a `setuid` executable that is included in the Hadoop distribution. The NodeManager uses this executable to launch and kill containers. The `setuid` executable switches to the user who has submitted the application and launches or kills the containers. For maximum security, this executor sets up restricted permissions and user/group ownership of local files and directories used by the containers such as the shared objects, jars, intermediate files, log files etc. Particularly note that, because of this, except the application owner and NodeManager, no other user can access any of the local files/directories including those localized as part of the distributed cache.【仅在GNU/Linux上支持，这个执行器要么作为提交应用程序的YARN用户(启用完全安全时)运行容器，要么在完全安全未启用时，作为专用用户(默认为nobody)运行容器。当启用完全安全时，此执行器要求在启动容器的集群节点上创建所有用户帐户。它使用在Hadoop发行版中的`setuid`可执行文件。NodeManager使用这个可执行文件来启动和终止容器。`setuid`可执行文件切换到提交了应用程序并启动或终止容器的用户。为了最大限度的安全，这个执行器为容器(如共享对象、jar、中间文件、日志文件等)所使用的本地文件和目录设置受限权限和用户/组所有权。特别要注意的是，由于这个原因，除了应用程序所有者和NodeManager之外，没有其他用户可以访问任何本地文件/目录，包括作为分布式缓存一部分进行本地化的文件/目录。】

> To build the LinuxContainerExecutor executable run:

要构建可执行的 LinuxContainerExecutor，运行:

    $ mvn package -Dcontainer-executor.conf.dir=/etc/hadoop/

> The path passed in `-Dcontainer-executor.conf.dir` should be the path on the cluster nodes where a configuration file for the `setuid` executable should be located. The executable should be installed in `$HADOOP_YARN_HOME/bin`.

在 `-Dcontainer-executor.conf.dir` 中传递的路径应该是集群节点上存放 `setuid` 可执行文件的路径。这个可执行文件应该安装在 `$HADOOP_YARN_HOME/bin` 目录下。

> The executable must have specific permissions: 6050 or `--Sr-s---` permissions user-owned by `root` (super-user) and group-owned by a special group (e.g. `hadoop`) of which the NodeManager Unix user is the group member and no ordinary application user is. If any application user belongs to this special group, security will be compromised. This special group name should be specified for the configuration property yarn.nodemanager.linux-container-executor.group in both `conf/yarn-site.xml` and `conf/container-executor.cfg`.

这个可执行文件必须具有特定的权限：6050 或 `--Sr-s---`，属于 root 用户(超级用户)，组属于一个特殊组(如`hadoop`)， NodeManager Unix 用户是该组的成员，而普通应用程序用户不是。

如果任何应用程序用户属于这个特殊组，安全性将受到威胁。应该在 `conf/yarn-site.xml` 和 `conf/container-executor.cfg` 中的 `yarn.nodemanager.linux-container-executor.group` 指定这个特殊组的名称。

> For example, let’s say that the NodeManager is run as user `yarn` who is part of the groups `users` and `hadoop`, any of them being the primary group. Let also be that `users` has both `yarn` and another user (application submitter) `alice` as its members, and alice does not belong to `hadoop`. Going by the above description, the `setuid/setgid` executable should be set 6050 or `--Sr-s---` with user-owner as `yarn` and group-owner as `hadoop` which has `yarn` as its member (and not `users` which has `alice` also as its member besides `yarn`).

例如，假设 NodeManager 作为用户 `yarn` 运行，`yarn` 是组 `users` 和 `hadoop` 的一部分，它们中的任何一个都是主组。

假设 `users` 同时拥有 `yarn` 和另一个用户(应用程序提交者)`alice`作为其成员，并且 `alice` 不属于 `hadoop`。

根据上面的描述，`setuid/setgid` 可执行文件应该设置为 6050 或 `--Sr-s---`，用户为 `yarn`，组为 `hadoop`，`hadoop` 将 `yarn` 作为其成员(而不是 `users`，它有 `alice` 作为其成员，除了`yarn`)。

> The LinuxTaskController requires that paths including and leading up to the directories specified in `yarn.nodemanager.local-dirs` and `yarn.nodemanager.log-dirs` to be set 755 permissions as described above in the table on permissions on directories.

LinuxTaskController 要求路径包括并指向 `yarn.nodemanager.local-dirs` 和 `yarn.nodemanager.log-dirs` 中指定的目录。并设置为 755 权限，如上面关于目录权限的表所示。

    conf/container-executor.cfg

> The executable requires a configuration file called `container-executor.cfg` to be present in the configuration directory passed to the mvn target mentioned above.

可执行文件需要一个 `container-executor.cfg` 配置文件存在于传递给上面提到的 mvn 目标的配置目录中。

> The configuration file must be owned by the user running NodeManager (user `yarn` in the above example), group-owned by anyone and should have the permissions 0400 or `r--------` .

配置文件必须由运行 NodeManager 的用户(上面示例中的用户`yarn`)拥有，组为任何人所有，应该具有权限 0400 或 `r--------`。

> The executable requires following configuration items to be present in the `conf/container-executor.cfg` file. The items should be mentioned as simple key=value pairs, one per-line:

可执行文件需要以下配置项出现在 `conf/container-executor.cfg` 文件中。这些项应该以简单的 key=value 对的形式出现，每行一个:

Parameter |  Value |  Notes
---|:---|:---
yarn.nodemanager.linux-container-executor.group | hadoop | Unix group of the NodeManager. The group owner of the container-executor binary should be this group. Should be same as the value with which the NodeManager is configured. This configuration is required for validating the secure access of the container-executor binary.【NodeManager的Unix组。container-executor binary的组所有者应该是这个组。应该与NodeManager配置的值相同。验证container-executor 的安全访问时需要此配置。】
banned.users  | hdfs,yarn,mapred,bin | Banned users.
allowed.system.users | foo,bar | Allowed system users.
min.user.id | 1000  | Prevent other super-users.

> To re-cap, here are the local file-sysytem permissions required for the various paths related to the LinuxContainerExecutor:

为了重新定义，以下是与 LinuxContainerExecutor 相关的各种路径所需的本地文件系统权限:

Filesystem | Path | User:Group  | Permissions
---|:---|:---|:---
local | container-executor          | root:hadoop | --Sr-s--*
local | conf/container-executor.cfg | root:hadoop | r-------*
local | yarn.nodemanager.local-dirs | yarn:hadoop | drwxr-xr-x
local | yarn.nodemanager.log-dirs   | yarn:hadoop | drwxr-xr-x

### 4.12、MapReduce JobHistory Server

Parameter |  Value |  Notes
---|:---|:---
`mapreduce.jobhistory.address` | MapReduce JobHistory Server host:port | Default port is 10020.
`mapreduce.jobhistory.keytab` | `/etc/security/keytab/jhs.service.keytab` Kerberos keytab file for the MapReduce JobHistory Server.
`mapreduce.jobhistory.principal` | `jhs/_HOST@REALM.TLD` | Kerberos principal name for the MapReduce JobHistory Server.

## 5、Multihoming

> Multihomed setups where each host has multiple hostnames in DNS (e.g. different hostnames corresponding to public and private network interfaces) may require additional configuration to get Kerberos authentication working. See [HDFS Support for Multihomed Networks](https://hadoop.apache.org/docs/r3.2.1/hadoop-project-dist/hadoop-hdfs/HdfsMultihoming.html)

在多宿主主机设置中，每个主机在 DNS 中有多个主机名(例如，不同的主机名对应于公共和私有网络接口)，可能需要额外的配置才能让 Kerberos 身份验证工作。

## 6、Troubleshooting

> Kerberos is hard to set up —and harder to debug. Common problems are

Kerberos很难设置，也更难调试。常见的问题是

- 网络和 DNS 配置。

- 主机上的 Kerberos 配置(`/etc/krb5.conf`)。

- Keytab 的创建和维护。

- 环境设置:JVM、用户登录、系统时钟等。

> Network and DNS configuration.
> Kerberos configuration on hosts (`/etc/krb5.conf`).
> Keytab creation and maintenance.
> Environment setup: JVM, user login, system clocks, etc.

> The fact that the error messages from the JVM are essentially meaningless does not aid in diagnosing and fixing such problems.

来自 JVM 的错误消息本质上是没有意义的，这一事实无助于诊断和修复此类问题。

> Extra debugging information can be enabled for the client and for any service

可以为客户端和任何服务启用额外的调试信息

> Set the environment variable HADOOP_JAAS_DEBUG to true.

设置 `HADOOP_JAAS_DEBUG` 为 true

    export HADOOP_JAAS_DEBUG=true

> Edit the log4j.properties file to log Hadoop’s security package at DEBUG level.

编辑 log4j.properties 文件在 DEBUG 级别记录 Hadoop 安全包。

    log4j.logger.org.apache.hadoop.security=DEBUG

> Enable JVM-level debugging by setting some system properties.

通过设置一些系统属性来启用 JVM 级调试。

    export HADOOP_OPTS="-Djava.net.preferIPv4Stack=true -Dsun.security.krb5.debug=true -Dsun.security.spnego.debug"

## 7、Troubleshooting with KDiag

> Hadoop has a tool to aid validating setup: `KDiag`

Hadoop有一个工具来帮助验证设置：KDiag

> It contains a series of probes for the JVM’s configuration and the environment, dumps out some system files (`/etc/krb5.conf`, `/etc/ntp.conf`), prints out some system state and then attempts to log in to Kerberos as the current user, or a specific principal in a named keytab.

它包含一系列针对 JVM 配置和环境的探测，转储一些系统文件(`/etc/krb5.conf`、`/etc/ntp.conf`)，然后打印一些系统状态，然后尝试以当前用户或指定的 keytab 中的特定主体登录 Kerberos。

> The output of the command can be used for local diagnostics, or forwarded to whoever supports the cluster.

该命令的输出可以用于本地诊断，也可以转发给支持集群的用户。

> The `KDiag` command has its own entry point; It is invoked by passing `kdiag` to `bin/hadoop` command. Accordingly, it will display the kerberos client state of the command used to invoke it.

KDiag 命令有自己的入口点；通过将 `kdiag` 传递给 `bin/hadoop` 命令来调用它。因此，它将显示用于调用它的命令的 kerberos 客户端状态。

    hadoop kdiag

> The command returns a status code of 0 for a successful diagnostics run. This does not imply that Kerberos is working —merely that the KDiag command did not identify any problem from its limited set of probes. In particular, as it does not attempt to connect to any remote service, it does not verify that the client is trusted by any service.

对于成功运行的诊断，该命令返回状态码 0。这并不意味着 Kerberos 正在工作，只是 KDiag命令没有从其有限的探测集中识别出任何问题。特别是，由于它不尝试连接到任何远程服务，因此它不验证客户端是否受任何服务的信任。

> If unsuccessful, exit codes are

如果不成功，则退出代码为

- -1：因为未知原因，命令执行失败

- 41：未授权(== HTTP’s 401)。KDiag 检测到一种导致 Kerberos 无法工作的情况。检查输出以识别问题。

> -1: the command failed for an unknown reason
> 41: Unauthorized (== HTTP’s 401). KDiag detected a condition which causes Kerberos to not work. Examine the output to identify the issue.

### 7.1、Usage

    KDiag: Diagnose Kerberos Problems
      [-D key=value] : Define a configuration option.
      [--jaas] : Require a JAAS file to be defined in java.security.auth.login.config.
      [--keylen <keylen>] : Require a minimum size for encryption keys supported by the JVM. Default value : 256.
      [--keytab <keytab> --principal <principal>] : Login from a keytab as a specific principal.
      [--nofail] : Do not fail on the first problem.
      [--nologin] : Do not attempt to log in.
      [--out <file>] : Write output to a file.
      [--resource <resource>] : Load an XML configuration resource.
      [--secure] : Require the hadoop configuration to be secure.
      [--verifyshortname <principal>]: Verify the short name of the specific principal does not contain '@' or '/'

#### 7.1.1、--jaas: Require a JAAS file to be defined in java.security.auth.login.config.

> If --jaas is set, the Java system property java.security.auth.login.config must be set to a JAAS file; this file must exist, be a simple file of non-zero bytes, and readable by the current user. More detailed validation is not performed.

如果设置了 `--jaas`，则 Java 系统属性 `java.security.auth.login.config` 必须设置为 JAAS 文件；该文件必须存在，是一个非零字节的简单文件，并且当前用户可读。不执行更详细的验证。

> JAAS files are not needed by Hadoop itself, but some services (such as Zookeeper) do require them for secure operation.

Hadoop 本身不需要 JAAS 文件，但是一些服务(如Zookeeper)为了安全操作需要它们。

#### 7.1.2、--keylen <length>: Require a minimum size for encryption keys supported by the JVM".

> If the JVM does not support this length, the command will fail.

如果 JVM 不支持这个长度，命令将失败。

> The default value is to 256, as needed for the AES256 encryption scheme. A JVM without the Java Cryptography Extensions installed does not support such a key length. Kerberos will not work unless configured to use an encryption scheme with a shorter key length.

默认值为 256，根据 AES256 加密方案的需要而定。没有安装 Java Cryptography Extensions 的 JVM 不支持这样的密钥长度。除非配置为使用密钥长度更短的加密方案，否则 Kerberos 无法工作。

#### 7.1.3、--keytab <keytab> --principal <principal>: Log in from a keytab.

> Log in from a keytab as the specific principal.

作为特定的主体从 keytab 登录。

- 1.该文件必须包含特定的主体，包括任何已命名的主机。也就是说，没有从 `_HOST` 到当前主机名的映射。

- 2.KDiag 将注销并尝试再次登录。这捕获了过去存在的 JVM 兼容性问题。(Hadoop 的 Kerberos 支持要求对特定于 jvm 的类使用/内省)。

> The file must contain the specific principal, including any named host. That is, there is no mapping from `_HOST` to the current hostname.

> KDiag will log out and attempt to log back in again. This catches JVM compatibility problems which have existed in the past. (Hadoop’s Kerberos support requires use of/introspection into JVM-specific classes).

#### 7.1.4、--nofail : Do not fail on the first problem

> KDiag will make a best-effort attempt to diagnose all Kerberos problems, rather than stop at the first one.

KDiag 将尽最大努力诊断所有 Kerberos 问题，而不是在第一个问题上就停止。

> This is somewhat limited; checks are made in the order which problems surface (e.g keylength is checked first), so an early failure can trigger many more problems. But it does produce a more detailed report.

这在某种程度上是有限的；检查是按照问题出现的顺序进行的(例如首先检查keyllength)，因此早期的故障可能会触发更多的问题。但它确实产生了一份更详细的报告。

#### 7.1.5、--nologin: Do not attempt to log in.

> Skip trying to log in. This takes precedence over the --keytab option, and also disables trying to log in to kerberos as the current kinited user.

跳过尝试登录。这优先于 `--keytab` 选项，并且还禁止尝试作为当前 kinited 用户登录到 kerberos。

> This is useful when the KDiag command is being invoked within an application, as it does not set up Hadoop’s static security state —merely check for some basic Kerberos preconditions.

当在应用程序中调用 KDiag 命令时，这非常有用，因为它不会设置 Hadoop 的静态安全状态，只检查一些基本的 Kerberos 先决条件。

#### 7.1.6、--out outfile: Write output to file.

    hadoop kdiag --out out.txt

> Much of the diagnostics information comes from the JRE (to stderr) and from Log4j (to stdout). To get all the output, it is best to redirect both these output streams to the same file, and omit the --out option.

许多诊断信息来自 JRE(到stderr)和 Log4j (到stdout)。

要获得所有输出，最好将这两个输出流重定向到同一个文件，并省略 `--out` 选项。

    hadoop kdiag --keytab zk.service.keytab --principal zookeeper/devix.example.org@REALM > out.txt 2>&1

> Even there, the output of the two streams, emitted across multiple threads, can be a bit confusing. It will get easier with practise. Looking at the thread name in the Log4j output to distinguish background threads from the main thread helps at the hadoop level, but doesn’t assist in JVM-level logging.

即使在那里，跨多个线程发出的两个流的输出也可能有点令人困惑。通过练习就会变得容易的。查看 Log4j 输出中的线程名来区分后台线程和主线程在 hadoop 级别上有帮助，但对 JVM 级别的日志记录没有帮助。

#### 7.1.7、--resource <resource> : XML configuration resource to load.

> To load XML configuration files, this option can be used. As by default, the core-default and core-site XML resources are only loaded. This will help, when additional configuration files has any Kerberos related configurations.

要加载 XML 配置文件，可以使用这个选项。默认情况下，只加载 core-default 和 core-site XML 资源。当其他配置文件具有任何 Kerberos 相关配置时，这将有所帮助。

    hadoop kdiag --resource hbase-default.xml --resource hbase-site.xml

> For extra logging during the operation, set the logging and HADOOP_JAAS_DEBUG environment variable to the values listed in “Troubleshooting”. The JVM options are automatically set in KDiag.

为了在操作期间进行额外的日志记录，将 logging 和 HADOOP_JAAS_DEBUG 环境变量设置为 Troubleshooting 中列出的值。JVM 选项在 KDiag 中自动设置。

#### 7.1.8、--secure: Fail if the command is not executed on a secure cluster.

> That is: if the authentication mechanism of the cluster is explicitly or implicitly set to “simple”:

即：如果集群的认证机制显式或隐式设置为“simple”:

```xml
<property>
  <name>hadoop.security.authentication</name>
  <value>simple</value>
</property>
```

> Needless to say, an application so configured cannot talk to a secure Hadoop cluster.

不用说，这样配置的应用程序无法与安全的 Hadoop 集群通信。

#### 7.1.9、--verifyshortname <principal>: validate the short name of a principal

> This verifies that the short name of a principal contains neither the "@" nor "/" characters.

这将验证主体的短名称既不包含 “@” 也不包含 “/” 字符。

### 7.2、Example

    hadoop kdiag \
      --nofail \
      --resource hdfs-site.xml --resource yarn-site.xml \
      --keylen 1024 \
      --keytab zk.service.keytab --principal zookeeper/devix.example.org@REALM

> This attempts to to perform all diagnostics without failing early, load in the HDFS and YARN XML resources, require a minimum key length of 1024 bytes, and log in as the principal zookeeper/devix.example.org@REALM, whose key must be in the keytab zk.service.keytab

这将尝试在不早失败的情况下执行所有诊断，加载 HDFS 和 YARN 的 XML 资源，要求至少 1024 字节的密钥长度，并以主 `zookeeper/devix.example.org@REALM` 登录，其密钥必须在 keytab `zk.service.keytab` 中。

## 8、References

- O’Malley O et al. [Hadoop Security Design](https://issues.apache.org/jira/secure/attachment/12428537/security-design.pdf)
- O’Malley O, [Hadoop Security Architecture](http://www.slideshare.net/oom65/hadoop-security-architecture)
- [Troubleshooting Kerberos on Java 7](http://docs.oracle.com/javase/7/docs/technotes/guides/security/jgss/tutorials/Troubleshooting.html)
- [Troubleshooting Kerberos on Java 8](http://docs.oracle.com/javase/8/docs/technotes/guides/security/jgss/tutorials/Troubleshooting.html)
- [Java 7 Kerberos Requirements](http://docs.oracle.com/javase/7/docs/technotes/guides/security/jgss/tutorials/Troubleshooting.html)
- [Java 8 Kerberos Requirements](http://docs.oracle.com/javase/8/docs/technotes/guides/security/jgss/tutorials/Troubleshooting.html)
- Loughran S., [Hadoop and Kerberos: The Madness beyond the Gate](https://steveloughran.gitbooks.io/kerberos_and_hadoop/content/)