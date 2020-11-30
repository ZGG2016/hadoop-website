# HDFS Snapshots

[TOC]

## 1、Overview

> HDFS Snapshots are read-only point-in-time copies of the file system. Snapshots can be taken on a subtree of the file system or the entire file system. Some common use cases of snapshots are data backup, protection against user errors and disaster recovery.

**HDFS 快照是文件系统某一时间点的只读副本**。

**可以在一个目录下或整个文件系统上获取快照**。

快照的一些常见用例是**数据备份、防止用户错误和灾难恢复**。

> The implementation of HDFS Snapshots is efficient:

HDFS 快照的实现是高效的:

- **创建快照是即时的**：复杂度为O(1)，不包括查找 inode 时间。

- **仅当对快照进行修改时才使用额外的内存**：O(M)，其中 M 是修改文件/目录的数量。

- **不会复制 datanodes 中的块**：快照文件记录了块的列表和文件大小。没有数据复制。

- **快照不会对常规的 HDFS 操作产生不利影响**：修改是按照逆时间顺序记录的，这样就可以直接访问到当前数据。快照数据是通过从当前数据中减去修改的数据来计算的。

> Snapshot creation is instantaneous: the cost is O(1) excluding the inode lookup time.

> Additional memory is used only when modifications are made relative to a snapshot: memory usage is O(M), where M is the number of modified files/directories.

> Blocks in datanodes are not copied: the snapshot files record the block list and the file size. There is no data copying.

> Snapshots do not adversely affect regular HDFS operations: modifications are recorded in reverse chronological order so that the current data can be accessed directly. The snapshot data is computed by subtracting the modifications from the current data.

### 1.1、Snapshottable Directories

> Snapshots can be taken on any directory once the directory has been set as snapshottable. A snapshottable directory is able to accommodate 65,536 simultaneous snapshots. There is no limit on the number of snapshottable directories. Administrators may set any directory to be snapshottable. If there are snapshots in a snapshottable directory, the directory can be neither deleted nor renamed before all the snapshots are deleted.

**只要目录被设置为可以拍摄快照，就可以在任意目录上拍摄快照。**

一个可拍快照目录能够容纳 65536 个同步快照。可拍快照的目录数量没有限制。

**管理员可以将任意目录设置为可拍快照的**。如果可拍快照的目录中有快照，则在所有快照删除之前，既不能删除该目录，也不能重命名该目录。

**嵌套的可拍快照的目录目前是不允许的**。换句话说，如果一个目录的一个祖先/后代目录是一个可拍快照目录，那么这个目录就不能设置为可拍快照。

> Nested snapshottable directories are currently not allowed. In other words, a directory cannot be set to snapshottable if one of its ancestors/descendants is a snapshottable directory.

### 1.2、Snapshot Paths

> For a snapshottable directory, the path component “.snapshot” is used for accessing its snapshots. Suppose /foo is a snapshottable directory, /foo/bar is a file/directory in /foo, and /foo has a snapshot s0. Then, the path /foo/.snapshot/s0/bar refers to the snapshot copy of /foo/bar. The usual API and CLI can work with the “.snapshot” paths. The following are some examples.

对于可拍快照目录，**路径组件 `.snapshot` 用于访问其快照**。

假设 `/foo` 是一个可拍快照目录，`/foo/bar` 是 `/foo` 中的文件或目录，`/foo`有一个快照 s0。然后是路径 `/foo/.snapshot/s0/bar`  是 `/foo/bar`的快照副本。

通常的 API 和 CLI 可以适合 `.snapshot` 路径。下面是一些例子。

> Listing all the snapshots under a snapshottable directory:

**在可拍快照目录下，列出所有的快照：**

	hdfs dfs -ls /foo/.snapshot

> Listing the files in snapshot s0:

**列出快照 s0 中的文件：**

	hdfs dfs -ls /foo/.snapshot/s0

> Copying a file from snapshot s0:

**从快照 s0 中复制一份文件：**

	hdfs dfs -cp -ptopax /foo/.snapshot/s0/bar /tmp

> Note that this example uses the preserve option to preserve timestamps, ownership, permission, ACLs and XAttrs.

注意：这个示例使用了 preserve 选项来保存时间戳、所有权、权限、ACLs 和 XAttrs。

## 2、Upgrading to a version of HDFS with snapshots

> The HDFS snapshot feature introduces a new reserved path name used to interact with snapshots: .snapshot. When upgrading from an older version of HDFS which does not support snapshots, existing paths named .snapshot need to first be renamed or deleted to avoid conflicting with the reserved path. See the upgrade section [in the HDFS user guide](https://hadoop.apache.org/docs/r3.2.1/hadoop-project-dist/hadoop-hdfs/HdfsUserGuide.html#Upgrade_and_Rollback) for more information.

HDFS 快照特性引入了一个用于与快照交互的新保留路径名：`.snapshot`。

**当从不支持快照的旧版本 HDFS 升级时，需要首先重命名或删除名为`.snapshot`的现有路径，以避免与保留路径冲突**。

有关更多信息，请参阅 HDFS 用户指南中的升级部分。

## 3、Snapshot Operations

### 3.1、Administrator Operations

> The operations described in this section require superuser privilege.

本节中描述的操作需要超级用户权限。

#### 3.1.1、Allow Snapshots

> Allowing snapshots of a directory to be created. If the operation completes successfully, the directory becomes snapshottable.

**允许目录可以创建快照的命令：**

	hdfs dfsadmin -allowSnapshot <path>

Arguments:

	path：The path of the snapshottable directory.

对应的 Java API 方法是 `HdfsAdmin` 中的 `void allowSnapshot(Path path)`

> See also the corresponding Java API void allowSnapshot(Path path) in HdfsAdmin.

#### 3.1.2、Disallow Snapshots

> Disallowing snapshots of a directory to be created. All snapshots of the directory must be deleted before disallowing snapshots.

**不允许目录被创建快照的命令：**

	hdfs dfsadmin -disallowSnapshot <path>

Arguments:

	path：The path of the snapshottable directory.

所有目录的快照必须在禁止前删除。

对应的 Java API 方法是 `HdfsAdmin` 中的 `void disallowSnapshot(Path path)`

> See also the corresponding Java API void disallowSnapshot(Path path) in HdfsAdmin.

### 3.2、User Operations

> The section describes user operations. Note that HDFS superuser can perform all the operations without satisfying the permission requirement in the individual operations.

本节描述用户操作。

注意，HDFS 超级用户可以在不满足单个操作的权限要求的情况下执行所有操作。

#### 3.2.1、Create Snapshots

> Create a snapshot of a snapshottable directory. This operation requires owner privilege of the snapshottable directory.

根据一个可拍快照的目录，**创建一个快照**。要求这个目录的所有者权限：

	hdfs dfs -createSnapshot <path> [<snapshotName>]

Arguments:

	path：The path of the snapshottable directory. 目录路径

	snapshotName：The snapshot name, which is an optional argument. When it is omitted, a default name is generated using a timestamp with the format "'s'yyyyMMdd-HHmmss.SSS", e.g. "s20130412-151029.033". 快照名，可选项，默认是使用时间戳

对应的 Java API 方法是 `FileSystem` 中的 `Path createSnapshot(Path path)` 和 `Path createSnapshot(Path path, String snapshotName)`

> See also the corresponding Java API Path createSnapshot(Path path) and Path createSnapshot(Path path, String snapshotName) in [FileSystem](https://hadoop.apache.org/docs/r3.2.1/api/org/apache/hadoop/fs/FileSystem.html) The snapshot path is returned in these methods.

#### 3.2.2、Delete Snapshots

> Delete a snapshot of from a snapshottable directory. This operation requires owner privilege of the snapshottable directory.

**删除快照**。要求这个目录的所有者权限：

	hdfs dfs -deleteSnapshot <path> <snapshotName>

Arguments:

	path：The path of the snapshottable directory.
	snapshotName：The snapshot name.

对应的 Java API 方法是 `FileSystem` 中的 `void deleteSnapshot(Path path, String snapshotName)`。

> See also the corresponding Java API void deleteSnapshot(Path path, String snapshotName) in [FileSystem](https://hadoop.apache.org/docs/r3.2.1/api/org/apache/hadoop/fs/FileSystem.html).

#### 3.2.3、Rename Snapshots

> Rename a snapshot. This operation requires owner privilege of the snapshottable directory.

**重命名快照**。要求这个目录的所有者权限：

	hdfs dfs -renameSnapshot <path> <oldName> <newName>

Arguments:

	path	The path of the snapshottable directory.
	oldName	The old snapshot name.
	newName	The new snapshot name.

对应的 Java API 方法是 `FileSystem` 中的 `void renameSnapshot(Path path, String oldName, String newName)`。

> See also the corresponding Java API void renameSnapshot(Path path, String oldName, String newName) in [FileSystem](https://hadoop.apache.org/docs/r3.2.1/api/org/apache/hadoop/fs/FileSystem.html).

#### 3.2.4、Get Snapshottable Directory Listing

> Get all the snapshottable directories where the current user has permission to take snapshtos.

**获取当前用户有权限去拍快照的所有目录**：

	hdfs lsSnapshottableDir

Arguments: none

对应的 Java API 方法是 `DistributedFileSystem` 中的 `SnapshottableDirectoryStatus[] getSnapshottableDirectoryListing()`。

> See also the corresponding Java API SnapshottableDirectoryStatus[] getSnapshottableDirectoryListing() in DistributedFileSystem.

#### 3.2.5、Get Snapshots Difference Report

> Get the differences between two snapshots. This operation requires read access privilege for all files/directories in both snapshots.

**获取两个快照间的差别**。这个操作要求对这两个快照下的文件/目录有读取权限：

	hdfs snapshotDiff <path> <fromSnapshot> <toSnapshot>

Arguments:

	path	The path of the snapshottable directory.
	fromSnapshot	The name of the starting snapshot.
	toSnapshot	The name of the ending snapshot.

> Note that snapshotDiff can be used to get the difference report between two snapshots, or between a snapshot and the current status of a directory. Users can use “.” to represent the current status.

注意：snapshotDiff 可用于获取两个快照之间的差异报告，或者快照与目录的当前状态之间的差异报告。用户可以使用 `.`，以代表现时的状况。

Results:

	+	The file/directory has been created.
	-	The file/directory has been deleted.
	M	The file/directory has been modified.
	R	The file/directory has been renamed.

RENAME 表示一个文件/目录已被重命名，但仍然位于相同的快照目录下。

**如果一个文件/目录被重命名到这个可拍快照目录之外，那么这个文件/目录被报告为删除**。

**从可拍快照目录之外，重命名的文件/目录被报告为新创建**。

> A RENAME entry indicates a file/directory has been renamed but is still under the same snapshottable directory. A file/directory is reported as deleted if it was renamed to outside of the snapshottble directory. A file/directory renamed from outside of the snapshottble directory is reported as newly created.

> The snapshot difference report does not guarantee the same operation sequence. For example, if we rename the directory “/foo” to “/foo2”, and then append new data to the file “/foo2/bar”, the difference report will be:

**快照差异报告不能保证相同的操作序列**。

例如，将目录 `/foo` 重命名为 `/foo2`，然后向文件 `/foo2/bar` 追加新数据，差异报告为:

	R. /foo -> /foo2
	M. /foo/bar

例如，重命名目录下的文件/目录的变化，在重命名之前，使用原始路径报告(上面例子中的`/foo/bar`)。

> I.e., the changes on the files/directories under a renamed directory is reported using the original path before the rename (“/foo/bar” in the above example).

对应的 Java API 方法是 `DistributedFileSystem` 中的 `SnapshotDiffReport getSnapshotDiffReport(Path path, String fromSnapshot, String toSnapshot)`。

> See also the corresponding Java API SnapshotDiffReport getSnapshotDiffReport(Path path, String fromSnapshot, String toSnapshot) in DistributedFileSystem.



---------------------------------------------------

```sh
# 允许创建
[root@zgg script]# hdfs dfsadmin -allowSnapshot /in
Allowing snapshot on /in succeeded

# 创建
[root@zgg script]# hdfs dfs -createSnapshot /in snaps01
Created snapshot /in/.snapshot/snaps01

[root@zgg script]# hadoop fs -ls /in                   
Found 9 items
-rw-r--r--   1 root supergroup         54 2020-11-23 20:39 /in/dedup.txt
-rw-r--r--   1 root supergroup        235 2020-11-18 17:04 /in/emp.lzo_deflate
-rw-r--r--   1 root supergroup        235 2020-11-18 17:04 /in/emp.txt
drwxr-xr-x   - root supergroup          0 2020-11-26 19:04 /in/ext03
drwxr-xr-x   - root supergroup          0 2020-11-26 19:00 /in/exttest
-rw-r--r--   1 root supergroup         48 2020-11-23 20:13 /in/secondsort.txt
drwxr-xr-x   - root supergroup          0 2020-11-22 17:29 /in/topk
drwxr-xr-x   - root supergroup          0 2020-11-22 17:43 /in/toptest
-rw-r--r--   1 root supergroup        185 2020-11-18 17:02 /in/user.avsc

# 重命名
[root@zgg script]# hdfs dfs -renameSnapshot /in snaps01 insnaps

# 列出能拍快照的目录
[root@zgg script]# hdfs lsSnapshottableDir
drwxr-xr-x 0 root supergroup 0 2020-11-30 15:40 1 65536 /in

# .snapshot目录下的内容
[root@zgg script]# hdfs dfs -ls /in/.snapshot 
Found 1 items
drwxr-xr-x   - root supergroup          0 2020-11-30 15:40 /in/.snapshot/insnaps

[root@zgg script]# hdfs dfs -ls /in/.snapshot/insnaps
Found 9 items
-rw-r--r--   1 root supergroup         54 2020-11-23 20:39 /in/.snapshot/insnaps/dedup.txt
-rw-r--r--   1 root supergroup        235 2020-11-18 17:04 /in/.snapshot/insnaps/emp.lzo_deflate
-rw-r--r--   1 root supergroup        235 2020-11-18 17:04 /in/.snapshot/insnaps/emp.txt
drwxr-xr-x   - root supergroup          0 2020-11-26 19:04 /in/.snapshot/insnaps/ext03
drwxr-xr-x   - root supergroup          0 2020-11-26 19:00 /in/.snapshot/insnaps/exttest
-rw-r--r--   1 root supergroup         48 2020-11-23 20:13 /in/.snapshot/insnaps/secondsort.txt
drwxr-xr-x   - root supergroup          0 2020-11-22 17:29 /in/.snapshot/insnaps/topk
drwxr-xr-x   - root supergroup          0 2020-11-22 17:43 /in/.snapshot/insnaps/toptest
-rw-r--r--   1 root supergroup        185 2020-11-18 17:02 /in/.snapshot/insnaps/user.avsc

[root@zgg script]# hadoop fs -rm /in/emp.lzo_deflate
Deleted /in/emp.lzo_deflate

[root@zgg script]# hdfs dfs -createSnapshot /in insnaps02
Created snapshot /in/.snapshot/insnaps02

# 两个快照间的差异
[root@zgg script]# hdfs snapshotDiff /in insnaps insnaps02
Difference between snapshot insnaps and snapshot insnaps02 under directory /in:
M       .
-       ./emp.lzo_deflate

[root@zgg script]# hadoop fs -ls /in/.snapshot
Found 2 items
drwxr-xr-x   - root supergroup          0 2020-11-30 15:40 /in/.snapshot/insnaps
drwxr-xr-x   - root supergroup          0 2020-11-30 15:46 /in/.snapshot/insnaps02

[root@zgg script]# hadoop fs -ls /in/.snapshot/insnaps02
Found 8 items
-rw-r--r--   1 root supergroup         54 2020-11-23 20:39 /in/.snapshot/insnaps02/dedup.txt
-rw-r--r--   1 root supergroup        235 2020-11-18 17:04 /in/.snapshot/insnaps02/emp.txt
drwxr-xr-x   - root supergroup          0 2020-11-26 19:04 /in/.snapshot/insnaps02/ext03
drwxr-xr-x   - root supergroup          0 2020-11-26 19:00 /in/.snapshot/insnaps02/exttest
-rw-r--r--   1 root supergroup         48 2020-11-23 20:13 /in/.snapshot/insnaps02/secondsort.txt
drwxr-xr-x   - root supergroup          0 2020-11-22 17:29 /in/.snapshot/insnaps02/topk
drwxr-xr-x   - root supergroup          0 2020-11-22 17:43 /in/.snapshot/insnaps02/toptest
-rw-r--r--   1 root supergroup        185 2020-11-18 17:02 /in/.snapshot/insnaps02/user.avsc

[root@zgg script]# hadoop fs -cat /in/.snapshot/insnaps02/dedup.txt
2017-7-1
2018-7-1
2018-7-1
2019-7-1
2019-7-1
2017-7-1

[root@zgg script]# hadoop fs -cat /in/dedup.txt
2017-7-1
2018-7-1
2018-7-1
2019-7-1
2019-7-1
2017-7-1

# 快照是只读的
[root@zgg data]# hadoop fs -appendToFile dedup2.txt /in/.snapshot/insnaps02/dedup.txt
appendToFile: Modification on a read-only snapshot is disallowed
```