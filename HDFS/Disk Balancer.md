# HDFS Disk Balancer

[TOC]

## 1、Overview

> Diskbalancer is a command line tool that distributes data evenly on all disks of a datanode. This tool is different from [Balancer](https://hadoop.apache.org/docs/r3.2.1/hadoop-project-dist/hadoop-hdfs/HdfsUserGuide.html#Balancer) which takes care of cluster-wide data balancing. Data can have uneven spread between disks on a node due to several reasons. This can happen due to large amount of writes and deletes or due to a disk replacement. This tool operates against a given datanode and moves blocks from one disk to another.

Diskbalancer 是一种命令行工具，**用于将数据均匀分布在 datanode 的所有磁盘上**。

这个工具**不同于 Balancer，后者负责集群范围内的数据平衡**。

由于一些原因，数据在一个节点上的磁盘间的分布可能不均匀。这种情况可能是由于大量的写和删除操作，或者由于更换了磁盘。该工具对给定的 datanode 进行操作，并将块从一个磁盘移动到另一个磁盘。

## 2、Architecture

> Disk Balancer operates by creating a plan and goes on to execute that plan on the datanode. A plan is a set of statements that describe how much data should move between two disks. A plan is composed of multiple move steps. A move step has source disk, destination disk and number of bytes to move. A plan can be executed against an operational data node. Disk balancer should not interfere with other processes since it throttles how much data is copied every second. Please note that disk balancer is not enabled by default on a cluster. To enable diskbalancer dfs.disk.balancer.enabled must be set to true in hdfs-site.xml.

磁盘平衡器的**操作方式是创建一个计划，然后在 datanode 上执行该计划**。

**计划是一组描述了在两个磁盘之间应该移动多少数据的语句**。计划是由多个移动步骤组成的。移动步骤具有源磁盘、目标磁盘和要移动的字节数。

一个计划可以在一个可操作的数据节点上执行。磁盘平衡器**不应该干扰其他进程，因为它会限制每秒钟复制的数据量**。

请注意，磁盘均衡器在集群中**默认是不启用的。要启用必须在 `hdfs-site.xml` 中设置 `dfs.disk.balancer.enabled=true`**。

## 3、Commands

> The following sections discusses what commands are supported by disk balancer and how to use them.

下面几节将讨论磁盘平衡器支持哪些命令，以及如何使用它们。

### 3.1、Plan

> The plan command can be run against a given datanode by running

**在给定的 datanode 上运行的 plan 命令**：

	hdfs diskbalancer -plan node1.mycluster.com

> The command accepts [Generic Options](https://hadoop.apache.org/docs/r3.2.1/hadoop-project-dist/hadoop-common/CommandsManual.html#Generic_Options).

该命令接受通用选项。

> The plan command also has a set of parameters that allows user to control the output and execution of the plan.

plan 命令还具有一组参数，允许用户控制计划的输出和执行。

COMMAND_OPTION       |     Description
---                  |:---
-out                 |  Allows user to control the output location of the plan file.【允许用户控制plan文件的输出位置】
-bandwidth           |  Since datanode is operational and might be running other jobs, diskbalancer limits the amount of data moved per second. This parameter allows user to set the maximum bandwidth to be used. This is not required to be set since diskBalancer will use the default bandwidth if this is not specified.【由于datanode处于可操作状态，并且可能正在运行其他作业，diskbalancer会限制每秒移动的数据量。用户可以通过该参数设置最大带宽。这是不需要设置的，因为如果没有指定，diskBalancer将使用默认带宽。】
-thresholdPercentage |	Since we operate against a snap-shot of datanode, the move operations have a tolerance percentage to declare success. If user specifies 10% and move operation is say 20GB in size, if we can move 18GB that operation is considered successful. This is to accommodate the changes in datanode in real time. This parameter is not needed and a default is used if not specified.【因为我们对datanode的快照进行操作，所以移动操作有一个容忍百分比来声明成功。如果用户指定了10%，移动操作的大小是20GB，如果我们移动18GB，那么这个操作就被认为是成功的。这是为了实时地适应datanode中的更改。不需要此参数，如果未指定则使用默认值。】
-maxerror            |  Max error allows users to specify how many block copy operations must fail before we abort a move step. Once again, this is not a needed parameter and a system-default is used if not specified.【允许用户指定在中止移动步骤之前，可以失败的块复制操作的次数。同样，这不是必需的参数，如果没有指定，则使用系统默认值。】
-v                   |  Verbose mode, specifying this parameter forces the plan command to print out a summary of the plan on stdout.【verbose模式，指定此参数将强制plan命令在标准输出上打印plan的摘要。】
-fs	                 |  - Specifies the namenode to use. if not specified default from config is used.【-使用的namenode。如果未指定，则使用使用配置中的默认项。】

> The plan command writes two output files. They are <nodename>.before.json which captures the state of the cluster before the diskbalancer is run, and <nodename>.plan.json.

plan 命令写入到两个输出文件。它们是 `<nodename>.before.json` 来捕获 diskbalancer 运行前的集群状态，以及 `<nodename>.plan.json`。	

### 3.2、Execute

> Execute command takes a plan command executes it against the datanode that plan was generated against.

**Execute 命令接受一个 plan 命令，在生成 plan 的 datanode 上执行它**。

	hdfs diskbalancer -execute /system/diskbalancer/nodename.plan.json

> This executes the plan by reading datanode’s address from the plan file. When DiskBalancer executes the plan, it is the beginning of an asynchronous process that can take a long time. So, query command can help to get the current status of execute command.

它通过从计划文件中读取 datanode 的地址来执行计划。

当 DiskBalancer 执行该计划时，它是一个需要很长时间的异步进程的开始。因此，query 命令可以帮助获取执行命令的当前状态。

COMMAND_OPTION | Description
---|:---
-skipDateCheck | Skip date check and force execute the plan.【跳过日期检查，强制执行计划。】

### 3.3、Query

> Query command gets the current status of the diskbalancer from a datanode.

**Query 命令一个 datanode 获取 diskbalancer 的当前状态**。

	hdfs diskbalancer -query nodename.mycluster.com

COMMAND_OPTION | Description
---|:---
-v | Verbose mode, Prints out status of individual moves【verbose模式，打印出单独移动的状态】

### 3.4、Cancel

> Cancel command cancels a running plan. Restarting datanode has the same effect as cancel command since plan information on the datanode is transient.

**Cancel 命令取消正在运行的计划**。

由于 datanode 上的 plan 信息是瞬态的，所以重启datanode 和取消命令的作用是一样的。

	hdfs diskbalancer -cancel /system/diskbalancer/nodename.plan.json

or

	hdfs diskbalancer -cancel planID -node nodename

> Plan ID can be read from datanode using query command.

Plan ID 可以通过 query 命令从 datanode 读取。

### 3.5、Report

> Report command provides detailed report of specified node(s) or top nodes that will benefit from running disk balancer. The node(s) can be specified by a host file or comma-separated list of nodes.

**Report 命令用于提供从运行硬盘均衡器受益的节点或前几个节点的详细报告**。可以通过主机文件或逗号分隔的节点列表指定节点。

	hdfs diskbalancer -fs http://namenode.uri -report -node <file://> | [<DataNodeID|IP|Hostname>,...]

or

	hdfs diskbalancer -fs http://namenode.uri -report -top topnum

## 4、Settings

> There is a set of diskbalancer settings that can be controlled via hdfs-site.xml

Setting                    |           Description
---                        |:---
dfs.disk.balancer.enabled  |   This parameter controls if diskbalancer is enabled for a cluster. if this is not enabled, any execute command will be rejected by the datanode.The default value is false.【控制集群是否启用diskbalancer。如果不启用，任何execute命令都会被datanode拒绝。默认值为false。】
dfs.disk.balancer.max.disk.throughputInMBperSec  |  This controls the maximum disk bandwidth consumed by diskbalancer while copying data. If a value like 10MB is specified then diskbalancer on the average will only copy 10MB/S. The default value is 10MB/S.【控制diskbalancer在复制数据时所消耗的最大磁盘带宽。如果指定了10MB这样的值，那么diskbalancer平均只会复制10MB/S。缺省值为10MB/S。】
dfs.disk.balancer.max.disk.errors  |  sets the value of maximum number of errors we can ignore for a specific move between two disks before it is abandoned. For example, if a plan has 3 pair of disks to copy between , and the first disk set encounters more than 5 errors, then we abandon the first copy and start the second copy in the plan. The default value of max errors is set to 5.【在放弃两个磁盘间的特定移动之前，可以忽略的最大错误数。例如，如果一个计划有3对磁盘可供复制，而第一个磁盘集遇到5个以上的错误，那么我们放弃第一个副本，开始计划中的第二个副本。最大错误数的默认值设置为5。】
dfs.disk.balancer.block.tolerance.percent  |  The tolerance percent specifies when we have reached a good enough value for any copy step. For example, if you specify 10% then getting close to 10% of the target value is good enough.【指定何时达到足够好的值来执行任何复制步骤。例如，如果您指定了10%，那么接近目标值的10%就足够了。】
dfs.disk.balancer.plan.threshold.percent   |  The percentage threshold value for volume Data Density in a plan. If the absolute value of volume Data Density which is out of threshold value in a node, it means that the volumes corresponding to the disks should do the balancing in the plan. The default value is 10.【计划中卷数据密度的百分比阈值。当某个节点上的卷数据密度绝对值超过阈值时，说明该磁盘对应的卷需要在计划中进行均衡。缺省值为10。】
dfs.disk.balancer.plan.valid.interval      |   Maximum amount of time disk balancer plan is valid. Supports the following suffixes (case insensitive): ms(millis), s(sec), m(min), h(hour), d(day) to specify the time (such as 2s, 2m, 1h, etc.). If no suffix is specified then milliseconds is assumed. Default value is 1d 【磁盘平衡器计划有效的最大时间。支持以下后缀(不区分大小写):ms(millis)、s(sec)、m(min)、h(hour)、d(day)来指定时间(如2s、2m、1h等)。如果没有指定后缀，则假定为毫秒。默认值为1d】

## 5、Debugging

> Disk balancer generates two output files. The nodename.before.json contains the state of cluster that we read from the namenode. This file contains detailed information about datanodes and volumes.

磁盘平衡器生成两个输出文件。`nodename.before.json` 包含我们从 namenode 读取的集群状态。该文件包含了 datanode 和卷的详细信息。

> if you plan to post this file to an apache JIRA, you might want to replace your hostnames and volume paths since it may leak your personal information.

如果你计划将此文件发布到 apache JIRA，你可能需要替换主机名和卷路径，因为它可能会泄露您的个人信息。

> You can also trim this file down to focus only on the nodes that you want to report in the JIRA.

还可以对该文件进行精简，以便只关注希望在 JIRA 中报告的节点。

> The nodename.plan.json contains the plan for the specific node. This plan file contains as a series of steps. A step is executed as a series of move operations inside the datanode.

`nodename.plan.json` 包含特定节点的计划。该计划文件包含一系列步骤。步骤在 datanode 内作为一系列移动操作执行。

> To diff the state of a node before and after, you can either re-run a plan command and diff the new nodename.before.json with older before.json or run report command against the node.

要区别节点前后的状态，您可以重新运行 plan 命令，并区别新的 `nodename.before.json` 与以前的 `before.json` 或对节点运行报告命令。

> To see the progress of a running plan, please run query command with option -v. This will print out a set of steps – Each step represents a move operation from one disk to another.

若要查看运行计划的进度，请执行带 -v 选项的 query 命令。这将打印出一组步骤：每个步骤代表从一个磁盘到另一个磁盘的移动操作。

> The speed of move is limited by the bandwidth that is specified. The default value of bandwidth is set to 10MB/sec. if you do a query with -v option you will see the following values.

移动的速度受到指定带宽的限制。缺省情况下，带宽为 10MB/秒。

如果使用 -v 选项进行查询，将看到以下值。

	  "sourcePath" : "/data/disk2/hdfs/dn",

	  "destPath" : "/data/disk3/hdfs/dn",

	  "workItem" :

	    "startTime" : 1466575335493,

	    "secondsElapsed" : 16486,

	    "bytesToCopy" : 181242049353,

	    "bytesCopied" : 172655116288,

	    "errorCount" : 0,

	    "errMsg" : null,

	    "blocksCopied" : 1287,

	    "maxDiskErrors" : 5,

	    "tolerancePercent" : 10,

	    "bandwidth" : 10

> source path - is the volume we are copying from.

source path：是我们要复制的卷。

> dest path - is the volume to where we are copying to.

dest path：我们要复制到的卷。

> start time - is current time in milliseconds.

start time：以毫秒为单位的当前时间。

> seconds elapsed - is updated whenever we update the stats. This might be slower than the wall clock time.

seconds elapsed：当我们更新统计时，就会更新。这可能比挂钟的时间要慢。

> bytes to copy - is number of bytes we are supposed to copy. We copy plus or minus a certain percentage. So often you will see bytesCopied – as a value lesser than bytes to copy. In the default case, getting within 10% of bytes to move is considered good enough.

bytes to copy：是我们应该复制的字节数。我们复制加或减一定的百分比。因此，你经常会看到 bytesCopied：一个小于要复制字节数的值。在默认情况下，将需要移动的字节数控制在10%以内就足够了。

> bytes copied - is the actual number of bytes that we moved from source disk to destination disk.

bytes copied：我们从源磁盘移动到目标磁盘的实际字节数。

> error count - Each time we encounter an error we will increment the error count. As long as error count remains less than max error count (default value is 5), we will try to complete this move. if we hit the max error count we will abandon this current step and execute the next step in the plan.

error count：每次遇到错误，都会增加错误计数。只要错误计数仍然小于最大错误计数(默认值是5)，我们将尝试完成这一步。如果我们达到最大错误数，我们将放弃当前步骤，并执行计划的下一步。

> error message - Currently a single string that reports the last error message. Older messages should be in the datanode log.

error message：当前一个单独的字符串，它报告最后一个错误消息。旧消息应该在datanode日志中。

> blocks copied - Number of blocks copied.

blocks copied：复制的块数。

> max disk errors - The configuration used for this move step. currently it will report the default config value, since the user interface to control these values per step is not in place. It is a future work item. The default or the command line value specified in plan command is used for this value.

max disk errors：这个移动步骤使用的配置。目前它将报告默认的配置值，因为每个步骤的控制这些值的用户界面还没有到位。它是未来的工作项。该值使用缺省值或plan命令中指定的命令行值。

> tolerance percent - This represents how much off we can be while moving data. In a busy cluster this allows admin to say, compute a plan, but I know this node is being used so it is okay if disk balancer can reach +/- 10% of the bytes to be copied.

tolerance percent：这表示在移动数据时我们所能做到的程度。在一个繁忙的集群中，这允许管理员说，计算一个计划，但我知道这个节点正在被使用，所以如果磁盘平衡器可以达到要复制的字节数的+/- 10%，这是没问题的。

> bandwidth - This is the maximum aggregate source disk bandwidth used by the disk balancer. After moving a block disk balancer computes how many seconds it should have taken to move that block with the specified bandwidth. If the actual move took less time than expected, then disk balancer will sleep for that duration. Please note that currently all moves are executed sequentially by a single thread.

bandwidth：这是磁盘均衡器使用的最大聚合源磁盘带宽。移动块之后，磁盘平衡器计算在指定带宽下移动该块应该花费多少秒。如果实际移动的时间比预期的短，那么磁盘平衡器将在那段时间内休眠。请注意，目前所有的移动都是由一个线程按顺序执行的。