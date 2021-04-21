# C API libhdfs

[TOC]

## 1、Overview

> libhdfs is a JNI based C API for Hadoop’s Distributed File System (HDFS). It provides C APIs to a subset of the HDFS APIs to manipulate HDFS files and the filesystem. libhdfs is part of the Hadoop distribution and comes pre-compiled in $HADOOP_HDFS_HOME/lib/native/libhdfs.so . libhdfs is compatible with Windows and can be built on Windows by running mvn compile within the hadoop-hdfs-project/hadoop-hdfs directory of the source tree.

libhdfs 是 HDFS 的基于 C API 的 JNI。**C APIs 作为 HDFS APIs 的子集 ，用来操作 HDFS 文件和文件系统**。

libhdfs 是 Hadoop 发行版的一部分，在 `$HADOOP_HDFS_HOME/lib/native/libhdfs.so` 中进行了预编译。

libhdfs 和 Windows 兼容，可以在 Windows 上的 `hadoop-hdfs-project/hadoop-hdfs` 目录中通过运行 mvn 编译器来构建。

## 2、The APIs

> The libhdfs APIs are a subset of the [Hadoop FileSystem APIs](https://hadoop.apache.org/docs/r3.2.1/api/org/apache/hadoop/fs/FileSystem.html).

libhdfs APIs 是 Hadoop 文件系统 APIs 的一个子集。

> The header file for libhdfs describes each API in detail and is available in $HADOOP_HDFS_HOME/include/hdfs.h.

libhdfs 的头文件描述了每个 API 的细节，**可在 `$HADOOP_HDFS_HOME/include/hdfs.h` 中找到**。 

## 3、A Sample Program

```c
#include "hdfs.h"

int main(int argc, char **argv) {

    hdfsFS fs = hdfsConnect("default", 0);
    const char* writePath = "/tmp/testfile.txt";
    hdfsFile writeFile = hdfsOpenFile(fs, writePath, O_WRONLY |O_CREAT, 0, 0, 0);
    if(!writeFile) {
          fprintf(stderr, "Failed to open %s for writing!\n", writePath);
          exit(-1);
    }
    char* buffer = "Hello, World!";
    tSize num_written_bytes = hdfsWrite(fs, writeFile, (void*)buffer, strlen(buffer)+1);
    if (hdfsFlush(fs, writeFile)) {
           fprintf(stderr, "Failed to 'flush' %s\n", writePath);
          exit(-1);
    }
    hdfsCloseFile(fs, writeFile);
}
```

## 4、How To Link With The Library

> See the CMake file for test_libhdfs_ops.c in the libhdfs source directory (hadoop-hdfs-project/hadoop-hdfs/src/CMakeLists.txt) or something like: gcc above_sample.c -I$HADOOP_HDFS_HOME/include -L$HADOOP_HDFS_HOME/lib/native -lhdfs -o above_sample

在 libhdfs 源目录 `hadoop-hdfs-project/hadoop-hdfs/src/CMakeLists.txt`中查看 `test_libhdfs_ops.c` 的 CMake 文件；

或者类似这样:`gcc above_sample.c -I$HADOOP_HDFS_HOME/include -L$HADOOP_HDFS_HOME/lib/native -lhdfs -o above_sample`

## 5、Common Problems

> The most common problem is the CLASSPATH is not set properly when calling a program that uses libhdfs. Make sure you set it to all the Hadoop jars needed to run Hadoop itself as well as the right configuration directory containing hdfs-site.xml. It is not valid to use wildcard syntax for specifying multiple jars. It may be useful to run `hadoop classpath --glob` or `hadoop classpath --jar <path>` to generate the correct classpath for your deployment. See [Hadoop Commands Reference](https://hadoop.apache.org/docs/r3.2.1/hadoop-project-dist/hadoop-common/CommandsManual.html#classpath) for more information on this command.

当使用 libhdfs 调用一个程序时，最常见的问题就是 CLASSPATH 没有正确设置。

**确保将 CLASSPATH 设置为了运行 hadoop 本身所有的 hadoop jars，和包含 hdfs-site.xml 的正确的配置目录**。

使用通配符语法来指定多个 jars 是无效的。

它是在运行 `hadoop classpath --glob` 或 `hadoop classpath --jar <path>` 为你的部署生成正确的类路径时是有用的。

## 6、Thread Safe

> libdhfs is thread safe.

libdhfs 是线程安全的。

> Concurrency and Hadoop FS “handles”

- 并发和 Hadoop FS “句柄”

	Hadoop FS 实现包括一个 FS 句柄缓存，它基于 namenode 的 URI 和用户连接进行缓存。

	因此，所有对 hdfsConnect 的调用将返回相同的句柄，但不同的用户调用 hdfsConnectAsUser 将返回不同的句柄。

	但是，由于 HDFS 客户端句柄是完全线程安全的，所以这对并发性没有影响。

> The Hadoop FS implementation includes an FS handle cache which caches based on the URI of the namenode along with the user connecting. So, all calls to hdfsConnect will return the same handle but calls to hdfsConnectAsUser with different users will return different handles. But, since HDFS client handles are completely thread safe, this has no bearing on concurrency.

> Concurrency and libhdfs/JNI

- 并发性和libhdfs / JNI

	libhdfs 对 JNI 的调用应该总是创建线程本地存储，所以(理论上)，libhdfs 应该像对 Hadoop FS 的底层调用一样是线程安全的。

> The libhdfs calls to JNI should always be creating thread local storage, so (in theory), libhdfs should be as thread safe as the underlying calls to the Hadoop FS.