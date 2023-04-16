---
title: slurm--故障排除指南
description: slurm中文翻译系列，机翻后纠正了一点，发现其他错误望指出
published: true
date: 2023-04-16T13:42:53.180Z
tags: slurm
editor: markdown
dateCreated: 2023-02-27T04:32:52.309Z
---

本指南旨在作为一个工具，帮助系统管理员或操作员排除Slurm故障和恢复服务。常问问题文件也可能被证明是有用的。


## Slurm没有响应

1. 执行 `"scontrol ping"`以确定主控制器和备份控制器是否有反应。
2. 如果它为你响应，这可能是一个网络或配置问题，具体到集群中的某些用户或节点。
3. 如果没有响应，直接登录到机器上，再试一次，排除网络和配置问题。
4. 如果仍然没有响应，通过执行 `"ps -el | grep slurmctld "`检查是否有一个活跃的slurmctld守护程序。
5. 如果slurmctld没有运行，重新启动它（通常以用户root身份使用`"/etc/init.d/slurm start "`命令）。你应该检查日志文件（`slurm.conf`文件中的`SlurmctldLog`）以了解它失败的原因。
6. 如果slurmctld正在运行但没有响应（这是一种非常罕见的情况），那么杀掉并重新启动它（通常以用户root身份使用`"/etc/init.d/slurm stop "`命令，然后`"/etc/init.d/slurm start"`）。
7. 如果它再次挂起，增加调试信息的粗略程度（在`slurm.conf`文件中增加`SlurmctldDebug`）并重新启动。再次检查日志文件，看是否有失败的迹象。
8. 如果它继续失败而没有显示失败模式，在不保留状态的情况下重新启动（通常以用户root身份使用`"/etc/init.d/slurm stop "`命令，然后`"/etc/init.d/slurm startclean"`）。注意：所有正在运行的作业和其他状态信息都将丢失。

## 作业没有被调度

这取决于Slurm使用的调度器。执行命令`"scontrol show config | grep SchedulerType "`来确定这一点。对于任何调度器，你可以使用 `"scontrol show job "`命令检查作业的优先级。

- 如果调度器类型是`builtin`，那么作业将按照给定分区的提交顺序来执行。即使有资源可以立即启动作业，它也会被推迟，直到没有先前提交的作业在等待。

- 如果调度器类型是`backfill`，那么作业一般会按照给定分区的提交顺序执行，但有一个例外：如果这样做不会延迟较早提交的作业的预期执行时间，那么较晚提交的作业将被提前启动。为了使回填调度有效，用户作业应该指定合理的时间限制。如果作业没有指定时间限制，那么所有的作业将收到相同的时间限制（与分区相关的时间限制），`backfill`调度作业的能力将受到限制。回填调度器不会改变所需或排除的节点的作业规范，所以指定节点的作业将大大降低回填调度的效果。更多细节请参见回填文档。

## 作业和节点被卡在`COMPLETING`状态下

这通常是由于与作业相关的不可杀死的进程。Slurm会继续尝试用`SIGKILL`终止进程，但有些作业可能会卡在执行I/O和不可杀死的状态。这通常是由于文件系统的问题，可以通过以下几种方式解决。

- 修复文件系统和/或重新启动节点。-或者-
- 将节点设置为停机状态，然后恢复服务（`"control update NodeName=<node> State=down Reason=hung_proc"` 和 `"control update NodeName=<node> State=resume"`）。这允许其他作业使用该节点，但留下不可杀死的进程。如果该进程完成了I/O，待定的SIGKILL应该立即终止它。-或者-
- 使用`UnkillableStepProgram`和`UnkillableStepTimeout`配置参数，通过发送电子邮件或重启节点，自动响应不能被杀死的进程。欲了解更多信息，请参见`slurm.conf`文档。

如果你的工作看起来不是因为文件系统问题而被卡住，可能需要一些调试才能找到原因。如果你能重现该行为，你可以将`SlurmdDebug`级别设置为 `"debug"`，并在你用来重现该问题的节点上重新启动slurmd。然后`slurmd.log`文件应该有更多的信息来帮助解决这个问题。查看`slurmctld.log`也可以提供一些线索。如果节点停止响应，你可能要研究一下原因，因为它们可能会阻止作业的清理，导致作业一直处于`COMPLETING`状态。当寻找连接问题时，相关的日志条目应该是这样的。

```
error: Nodes node[00,03,25] not responding
Node node00 now responding
```

## 节点被设置为下降状态
1. 使用 `"scontrol show node <name>"`命令检查节点停机的原因。这将显示节点被设置为停机的原因和发生的时间。如果与`slurm.conf`文件中指定的参数相比，磁盘空间、内存空间等不足，那么要么修复节点，要么改变`slurm.conf`。
2. 如果原因是 `"Not responding"`，那么使用 `"ping <address>"`命令检查控制机和DOWN节点之间的通信，并确保指定`slurm.conf`中配置的`NodeAddr`值。如果ping失败了，那么就修改`slurm.conf`中的网络或地址。
3. 接下来，登录到一个Slurm认为处于下降状态的节点，用 `"ps -el | grep slurmd "`命令检查slurmd守护进程是否正在运行。如果slurmd没有运行，重新启动它（通常以用户root身份使用命令`"/etc/init.d/slurm start"`）。你应该检查日志文件（`slurm.conf`文件中的`SlurmdLog`）以了解它失败的原因。你可以通过在感兴趣的节点上执行 `"scontrol show slurmd "`命令来获得运行中的slurmd守护进程的状态。检查 `"Last slurmctld msg time "`的值以确定slurmctld是否能够与slurmd通信。
4. 如果slurmd正在运行但没有响应（一种非常罕见的情况），那么就杀掉并重新启动它（通常以用户root身份使用`"/etc/init.d/slurm stop "`命令，然后`"/etc/init.d/slurm start"`）。
5. 如果仍然没有响应，再试一次以排除网络和配置问题。
6. 如果仍然没有反应，增加调试信息的粗略程度（在`slurm.conf`文件中增加`SlurmdDebug`）并重新启动。再次检查日志文件，看是否有失败原因的提示。
7. 如果仍然没有反应而没有显示故障模式，在不保留状态的情况下重新启动（通常以用户root身份使用`"/etc/init.d/slurm stop "`然后`"/etc/init.d/slurm startclean "`命令）。注意：该节点上的所有作业和其他状态信息都将丢失。

## 网络和配置问题

1. 检查控制器和/或slurmd日志文件（`slurmctldLog`和`slurmdLog`在slurm.conf文件中），以了解它失败的原因。
2. 检查遇到问题的节点上的`slurm.conf`和凭证文件是否一致。
3. 如果这是用户特有的问题，检查用户是否在控制器计算机以及计算节点上配置。该用户不需要能够登录，但他的用户ID必须存在。
4. 检查所有节点上是否存在兼容的Slurm版本（执行 `"sinfo -V"` 或 `"rpm -qa | grep slurm"`）。Slurm的版本号包含三个期数分开的数字，代表Slurm的主要版本和维护版本级别。前两个部分结合在一起代表主要版本，并与该主要版本的年份和月份相符。版本中的第三个数字指定了一个特定的维护级别：年.月.维护-版本（例如，17.11.5是主要的Slurm版本17.11，以及维护版本5）。因此，17.11.x版本最初是在2017年11月发布的。Slurm守护进程将支持前两个主要版本的RPC和状态文件（例如，17.11.x版本的SlurmDBD将支持版本为17.11.x、17.02.x或16.05.x的slurmctld守护进程和命令）。