# Commands Reference

[TOC]

#### job

> Command to interact with Map Reduce Jobs.

和 Map Reduce Jobs 交互的命令。

Usage: 

	mapred job | [GENERIC_OPTIONS] | [-submit <job-file>] | [-status <job-id>] | [-counter <job-id> <group-name> <counter-name>] | [-kill <job-id>] | [-events <job-id> <from-event-#> <#-of-events>] | [-history [all] <jobHistoryFile|jobId> [-outfile <file>] [-format <human|json>]] | [-list [all]] | [-kill-task <task-id>] | [-fail-task <task-id>] | [-set-priority <job-id> <priority>] | [-list-active-trackers] | [-list-blacklisted-trackers] | [-list-attempt-ids <job-id> <task-type> <task-state>] [-logs <job-id> <task-attempt-id>] [-config <job-id> <file>]

COMMAND_OPTION  |  Description
---|:---
-submit job-file |	 Submits the job.  【提交job】
-status job-id   |	 Prints the map and reduce completion percentage and all job counters. 【打印map和reduce完成度和所有job的统计信息】
-counter job-id group-name counter-name	 | Prints the counter value.
-kill job-id  |  Kills the job.
-events job-id from-event-# #-of-events | Prints the events’ details received by jobtracker for the given range. 【打印jobtracker接收的事件详细信息】
-history [all] jobHistoryFilejobId [-outfile file] [-format humanjson]|	Prints job details, failed and killed task details. More details about the job such as successful tasks, task attempts made for each task, task counters, etc can be viewed by specifying the [all] option. An optional file output path (instead of stdout) can be specified. The format defaults to human-readable but can also be changed to JSON with the [-format] option.【打印job的详细信息，失败的和杀死的任务详细信息。但更多的是job的详细信息，如成功的任务，每个任务尝试的次数，任务计数器等，可以指定all选项被浏览。可以指定一个一个可选的文件输出路径(而不是标准输出)，格式是可读的，也可以通过-format指定为JSON】
-list [all]       |Displays jobs which are yet to complete. -list all displays all jobs. 【列出所有还未完成的job。-list all则列出所有job。】
-kill-task task-id  | Kills the task. Killed tasks are NOT counted against failed attempts.【杀死指定任务。杀死的任务不计入失败的尝试。】
-fail-task task-id  |Fails the task. Failed tasks are counted against failed attempts. 【fail指定任务。杀死的任务不计入失败的尝试。】
-set-priority job-id priority | Changes the priority of the job. Allowed priority values are `VERY_HIGH, HIGH, NORMAL, LOW, VERY_LOW`【修改job优先级】
-list-active-trackers | List all the active NodeManagers in the cluster.【列出集群中所有的活跃NodeManagers】
-list-blacklisted-trackers  | List the black listed task trackers in the cluster. This command is not supported in MRv2 based cluster.
-list-attempt-ids job-id task-type task-state  |  List the attempt-ids based on the task type and the status given. Valid values for task-type are `REDUCE, MAP`. Valid values for task-state are `running, pending, completed, failed, killed`.【根据给定的任务类型和状态列出attempt-ids】
-logs job-id task-attempt-id  | Dump the container log for a job if taskAttemptId is not specified, otherwise dump the log for the task with the specified taskAttemptId. The logs will be dumped in system out.【如果未指定taskAttemptId，则转储job的容器日志，否则转储指定的taskAttemptId的任务的日志。日志将转储到系统输出中。】
-config job-id file  | Download the job configuration file.【下载job配置文件】