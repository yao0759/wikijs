---
title: slurm--网络配置指南
description: slurm中文翻译系列，机翻后纠正了一点，发现其他错误望指出，来源：https://github.com/SchedMD/slurm/blob/master/doc/html/network.shtml
published: true
date: 2023-03-19T16:27:15.626Z
tags: slurm
editor: markdown
dateCreated: 2022-12-30T14:45:27.469Z
---

## 概述

在Slurm集群中，有很多组件需要能够相互通信。有些站点有安全要求，不能打开机器之间的所有通信，需要有选择地打开必要的端口。本文件将介绍不同的组件需要怎样才能相互交流。

下面是一个相当典型的集群图，`slurmctld`和`slurmdbd`在不同的机器上。在较小的集群中，MySQL可以和`slurmdbd`运行在同一台机器上，但在大多数情况下，最好是让它运行在一台专门的机器上。 `slurmd`运行在计算节点上，客户端命令可以在你选择的机器上安装和运行。

![image-20221228220925705](https://yhblog-1254039996.cos.ap-guangzhou.myqcloud.com/img-blog/image-20221228220925705.png)

## slurmctld的通讯方式

slurmctld用于监听传入请求的默认端口是`6817`，这个端口可以通过`slurm.conf`修改`SlurmctldPort `参数改变。Slurmctld在该端口监听传入的请求，并在请求者打开的同一连接上作出回应。

运行`slurmctld`的机器也需要能够建立对外的连接，它需要在默认的`6819`端口与`slurmdbd`进行通信。它还需要与计算节点上的`slurmd`进行通信，默认端口为`6818`。

默认情况下，slurmctld会监听IPv4流量。通过在`slurm.conf`的`CommunicationParameters`中加入`EnableIPv6`，可以启用IPv6通信。在启用IPv6后，你可以通过在`CommunicationParameters`中加入`DisableIPv4`来禁用IPv4。这些设置必须在`slurmdbd.conf`和`slurm.conf`中匹配。

## slurmdbd的通信

`slurmdbd`用于监听传入请求的默认端口是`6819`，这个端口可以通过`slurmdbd.conf`上的`SlurmctldPort `参数改变。Slurmdbd在该端口监听传入的请求，并在请求者打开的同一连接上进行响应。

运行`slurmdbd`的机器需要能够到达MySQL或MariaDB服务器，默认端口为`3306`。这个端口可以通过slurmdbd.conf上的`StoragePort `参数来改变。它还需要能够启动与`slurmctld`的连接，默认端口为`6819`。

默认情况下，`slurmdbd`将监听IPv4流量。通过在`slurmdbd.conf`的`CommunicationParameters`中加入`EnableIPv6`，可以启用IPv6通信。在启用IPv6后，你可以通过在`CommunicationParameters`中加入`DisableIPv4`来禁用IPv4。这些设置必须在`slurmdbd.conf`和`slurm.conf`中匹配。

## slurmd的通信

`slurmd`用于监听来自`slurmctld`的传入请求的默认端口是`6818`，这个端口可以通过`slurm.conf`上的`SlurmdPort `参数来改变。

运行`srun`的机器也使用一系列的端口，以便能够与`slurmstepd`通信。默认情况下，这些端口是从短暂的端口范围中随机选择的，但是你可以使用`SrunPortRange`来指定一个可以从中选择的端口范围。这对于在防火墙后面的登录节点是必要的。

运行`slurmd`的机器需要能够在默认的`6817`端口与`slurmctld`建立连接。

默认情况下，`slurmd`通过IPv4进行通信。由于`slurm.conf`参数也会影响`slurmd`守护进程，请参见`slurmctld`部分，以了解如何改变这一点。

## 客户端命令的通信

大多数客户端命令默认会在`6817`端口与`slurmctld`进行通信（关于如何改变这一点，请参见`slurmctld`部分），以获得它们需要的信息。这包括以下命令。

- salloc
- sacctmgr
- sbatch
- sbcast
- scancel
- scontrol
- sdiag
- sinfo
- sprio
- squeue
- sshare
- sstat
- strigger
- sview

还有一些命令与`slurmdbd`直接通信，默认端口为`6819`，下面的命令从`slurmdbd`获取信息：

- sacct
- sacctmgr
- sreport

当用户使用`srun`启动一个作业时，必须有一个从调用`srun`的机器到作业分配的节点的通信路径。通信遵循下面的顺序。

1. srun向slurmctld发送作业分配请求
2. slurmctld批准分配并返回详细信息
3. srun向slurmctld发送步骤创建请求
4. slurmctld用步骤凭证进行响应
5. srun为I/O打开套接字
6. srun将带有任务信息的凭证转发给slurmd
7. slurmd根据需要转发请求（按扇出）。
8. slurmd forks/execs slurmstepd
9. slurmstepd连接I/O并启动任务
10. 在任务终止时，slurmstepd会通知srun
11. srun通知slurmctld任务终止
12. slurmctld通过slurmd验证所有进程的终止，并为下一个作业释放资源

![image-20221228221510376](https://yhblog-1254039996.cos.ap-guangzhou.myqcloud.com/img-blog/image-20221228221510376.png)

## 与多个控制器的通信

你可以配置一个次要的`slurmctld`和或`slurmdbd`，作为主控制器发生故障时的后备。所涉及的端口不会改变，但有额外的通信路径需要考虑到。客户端命令需要能够到达运行`slurmctld`的两台机器，以及运行`slurmdbd`的两台机器。`slurmctld`的两个实例都需要能够到达`slurmdbd`的两个实例，每个`slurmdbd`都需要能够到达MySQL服务器。

![image-20221228221603538](https://yhblog-1254039996.cos.ap-guangzhou.myqcloud.com/img-blog/image-20221228221603538.png)

## 与多个集群的通信

在多个slurmctld实例共享同一个slurmdbd的环境中，你可以将每个集群配置成独立的，并允许用户指定一个集群来提交他们的作业。不同守护进程使用的端口不会改变，但所有slurmctld实例都需要能够与同一个slurmdbd实例通信。你可以在多集群操作文档中阅读更多关于多集群配置的内容。

![image-20221228221705657](https://yhblog-1254039996.cos.ap-guangzhou.myqcloud.com/img-blog/image-20221228221705657.png)

## federation中的通信

Slurm还提供了在多个集群之间以点对点方式安排作业的能力，允许作业首先在有可用资源的集群上运行。这与多集群配置在通信需求上的区别在于，slurmctld的两个实例需要能够相互通信。在文档中有更多关于使用federation的细节。

![image-20221228221747207](https://yhblog-1254039996.cos.ap-guangzhou.myqcloud.com/img-blog/image-20221228221747207.png)

## 与IPv6的通信

slurmctld、slurmdbd和slurmd守护进程默认使用IPv4通信，但它们可以被配置为使用IPv6。这可以通过在slurm.conf和slurmdbd.conf中设置`CommunicationParameters=EnableIPv6`，然后重新启动所有的守护进程来处理。在这种模式下，slurmd可以通过IPv4或IPv6运行。可以通过设置`CommunicationParameters=EnableIPv6,DisableIPv4`来禁用IPv4。在这个模式下，所有的东西都必须有一个有效的IPv6地址，否则连接会失败。

slurmctld希望一个节点能映射到一个IP地址（这将是用getaddrinfo()查找节点的IP时返回的第一个地址）。如果你在一个现有的集群上启用了IPv6，并且节点有IPv6地址，你必须重新启动slurmd守护进程以建立IPv6的通信。

在 /etc/gai.conf 中出现的优先级 ::fff:0:0/96 100 将导致 IPv4 地址在 IPv6 地址之前被返回。这可能会导致这样一种情况：你已经为 Slurm 启用了 IPv6，但仍然看到节点在用 IPv4 通信。如果对哪个地址被使用感到困惑，你可以调用 scontrol setdebugflags +NET 来在 slurmctld.log 中启用网络相关的调试日志。

如果启用了 IPv4 和 IPv6，环回接口可能仍然解析为 127.0.0.1。这不一定说明有问题。