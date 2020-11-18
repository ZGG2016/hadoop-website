# Hadoop Commands Guide

[TOC]

#### appendToFile

Usage: <font color="red">hadoop fs -appendToFile <localsrc> ... <dst></font>

将本地文件系统的单个文件或多个文件，追加到目标文件系统。

也可以从标准输入中读取，再追加到目标文件系统。

<font color="grey">Append single src, or multiple srcs from local file system to the destination file system. Also reads input from stdin and appends to destination file system.</font>

	hadoop fs -appendToFile localfile /user/hadoop/hadoopfile
		
	hadoop fs -appendToFile localfile1 localfile2 /user/hadoop/hadoopfile
		
	hadoop fs -appendToFile localfile hdfs://nn.example.com/hadoop/hadoopfile
		
	hadoop fs -appendToFile - hdfs://nn.example.com/hadoop/hadoopfile Reads the input from stdin.

Exit Code:

	Returns 0 on success and 1 on error.

```sh
[root@zgg data]# cat wc2.txt
aa cc bb aa cc dd
[root@zgg data]# hadoop fs -appendToFile wc2.txt /in/wc.txt
[root@zgg data]# hadoop fs -cat /in/wc.txt                 
hello hadoop spark hello flink hadoop hadoop
aa cc bb aa cc dd
```

#### cat

Usage: <font color="red">hadoop fs -cat [-ignoreCrc] URI [URI ...]</font>

将源路径复制到标准输出

<font color="grey">Copies source paths to stdout.</font>

Options：

- -ignoreCrc: 取消 checkshum 验证功能。

<font color="grey">The -ignoreCrc option disables checkshum verification.</font>

Example:

	hadoop fs -cat hdfs://nn1.example.com/file1 hdfs://nn2.example.com/file2

	hadoop fs -cat file:///file3 /user/hadoop/file4

Exit Code:

	Returns 0 on success and -1 on error.

#### rm

Usage: <font color="red">hadoop fs -rm [-f] [-r |-R] [-skipTrash] [-safely] URI [URI ...]</font>

删除指定文件。

如果启用了 trash ，文件系统会把删除的文件移至 trash 目录。

当前 trash 默认是关闭的，可以设置 `fs.trash.interval`(core-site.xml) 为一个大于0的值来启用。

<font color="grey">Delete files specified as args.</font>

<font color="grey">If trash is enabled, file system instead moves the deleted file to a trash directory (given by [FileSystem#getTrashRoot](https://hadoop.apache.org/docs/r3.2.1/api/org/apache/hadoop/fs/FileSystem.html)).</font>

<font color="grey">Currently, the trash feature is disabled by default. User can enable trash by setting a value greater than zero for parameter fs.trash.interval (in core-site.xml).</font>

<font color="grey">See [expunge](https://hadoop.apache.org/docs/r3.2.1/hadoop-project-dist/hadoop-common/FileSystemShell.html#expunge) about deletion of files in trash.</font>

Options:

- -f: 如果文件不存在，则不显示诊断信息或修改退出状态，以反映错误。

- -R: 递归的删除目录和下面的任意内容

 - -r: 同 -R

 - -skipTrash: 如果启用了 trash ，删除文件时直接删除。

 - -safely: 当删除的目录包含的文件总数量超过 `hadoop.shell.delete.limit.num.files`(in core-site.xml, default: 100)时，删除前会进行安全验证。它可以与`-skipTrash`一起使用，以防止意外删除大型目录。在确认之前递归遍历大目录以计算要删除的文件数量时，会出现延迟。

<font color="grey">The -f option will not display a diagnostic message or modify the exit status to reflect an error if the file does not exist.
	
The -R option deletes the directory and any content under it recursively.
	
The -r option is equivalent to -R.
	
The -skipTrash option will bypass trash, if enabled, and delete the specified file(s) immediately. This can be useful when it is necessary to delete files from an over-quota directory.
	
The -safely option will require safety confirmation before deleting directory with total number of files greater than hadoop.shell.delete.limit.num.files (in core-site.xml, default: 100). It can be used with -skipTrash to prevent accidental deletion of large directories. Delay is expected when walking over large directory recursively to count the number of files to be deleted before the confirmation.</font>

Example:

	hadoop fs -rm hdfs://nn.example.com/file /user/hadoop/emptydir

Exit Code:

	Returns 0 on success and -1 on error.

#### text

Usage: <font color="red">hadoop fs -text <src></font>

获取源文件并以文本格式输出该文件。

允许的格式是 zip 和 TextRecordInputStream。

<font color="red">Takes a source file and outputs the file in text format. The allowed formats are zip and TextRecordInputStream.</font>