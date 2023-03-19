---
title: slurm--拓扑指南
description: 
published: true
date: 2023-03-19T15:34:03.162Z
tags: slurm
editor: markdown
dateCreated: 2023-03-19T15:34:03.162Z
---


Slurm可以被配置为支持拓扑感知的资源分配，以优化工作性能。Slurm支持几种操作模式，一种是在具有三维环形互连的系统上优化性能，另一种是分层互连的模式。分层操作模式同时支持胖树或蜻蜓网络，使用的算法略有不同。

Slurm的本地资源选择模式是将节点视为一个一维阵列。工作在最适合的基础上被分配资源。对于较大的工作，这使分配给工作的连续节点集的数量最小。

## 三维拓扑结构
一些较大的计算机依赖于三维环形互连。Cray XT和XE系统也有三维环形互连，但不要求作业在相邻节点执行。在这些系统上，Slurm只需要为作业分配网络上附近的资源。Slurm使用希尔伯特曲线将节点从三维空间映射到一维空间来完成这一工作。因此，Slurm的本地最佳匹配算法能够实现工作的高度定位。

## 分层网络
Slurm也可以被配置为在分层网络上为作业分配资源，以尽量减少网络争用。基本算法是确定层次结构中能够满足作业请求的最低级别的交换机，然后使用最佳匹配算法在其底层叶子交换机上分配资源。使用这种逻辑需要TopologyPlugin=topology/tree的配置设置。

注意，slurm在当前可用的资源上使用最佳匹配算法。这可能会导致分配的交换机数量超过了最佳数量。用户可以使用salloc、sbatch和srun命令中的--switches选项，要求作业的最大叶子开关数，以及愿意等待该数量的最大时间。这些参数也可以通过scontrol和squeue命令对待处理的作业进行修改。

在未来的某个时候，Slurm代码可能会被提供来直接收集网络拓扑信息。现在，网络拓扑信息必须包含在topology.conf配置文件中，如下面的例子所示。第一个例子描述了一个三层交换机，其中每个交换机有两个子节点。请注意，SwitchName值是任意的，只用于记账，但每一行都必须指定一个名称。叶子交换机的描述包含一个SwitchName字段和一个Nodes字段，用于识别连接到该交换机的节点。更高级别的交换机描述包含一个SwitchName字段和一个Switches字段，用于识别子交换机。使用Slurm的hostlist表达式解析器，所以节点和交换机的名字不需要是连续的（例如 "Nodes=tux[0-3,12,18-20]"和 "Switches=s[0-2,4-8,12]"会解析的很好）。

一个可选的LinkSpeed选项可以用来表示链接的相对性能。所用的单位是任意的，而且这一信息目前还没有被使用。它可能在将来被用来优化资源分配。

第一个例子显示了一个八节点集群的拓扑结构，其中所有的交换机都只有两个子节点，如图所示（不是一个非常现实的配置，但对一个例子很有用）。

```
# topology.conf
# Switch Configuration
SwitchName=s0 Nodes=tux[0-1]
SwitchName=s1 Nodes=tux[2-3]
SwitchName=s2 Nodes=tux[4-5]
SwitchName=s3 Nodes=tux[6-7]
SwitchName=s4 Switches=s[0-1]
SwitchName=s5 Switches=s[2-3]
SwitchName=s6 Switches=s[4-5]
```

![image-20220914225313815](https://yhblog-1254039996.cos.ap-guangzhou.myqcloud.com/img-blog/image-20220914225313815.png)

下一个例子是一个有两层的网络，每个交换机有四个连接。

```
# topology.conf
# Switch Configuration
SwitchName=s0 Nodes=tux[0-3]   LinkSpeed=900
SwitchName=s1 Nodes=tux[4-7]   LinkSpeed=900
SwitchName=s2 Nodes=tux[8-11]  LinkSpeed=900
SwitchName=s3 Nodes=tux[12-15] LinkSpeed=1800
SwitchName=s4 Switches=s[0-3]  LinkSpeed=1800
SwitchName=s5 Switches=s[0-3]  LinkSpeed=1800
SwitchName=s6 Switches=s[0-3]  LinkSpeed=1800
SwitchName=s7 Switches=s[0-3]  LinkSpeed=1800
```

![image-20220914225348561](C:/Users/yaohu/AppData/Roaming/Typora/typora-user-images/image-20220914225348561.png)

作为一个实际问题，列出每一个交换机连接肯定会导致Slurm的调度算法更慢，以优化作业放置。应用程序的性能可能从这种优化中获得很少的好处。列出叶子交换机与它们的节点加上一个顶层交换机，应该会给应用程序和Slurm带来良好的性能。前面的例子可能会被配置成这样。

```
# topology.conf
# Switch Configuration
SwitchName=s0 Nodes=tux[0-3]
SwitchName=s1 Nodes=tux[4-7]
SwitchName=s2 Nodes=tux[8-11]
SwitchName=s3 Nodes=tux[12-15]
SwitchName=s4 Switches=s[0-3]
```

注意，可以使用缺乏共同父级交换机上的计算节点，但没有作业会跨越没有共同父级的叶子交换机（除非使用TopologyParam=TopoOptional选项）。例如，从上述 topology.conf 文件中删除 "SwitchName=s4 Switches=s[0-3]"一行是合法的。在这种情况下，任何作业都不会在任何单一的叶子交换机上跨越超过四个计算节点。如果想把多个物理集群作为一个单一的逻辑集群在一个slurmctld守护程序的控制下进行调度，这种配置可能很有用。

如果你的节点在不同的网络中，并且在你的topology.conf文件中与独特的交换机相关联，你就有可能陷入作业无法运行的情况。如果一个作业请求处于不同网络中的节点，无论是直接请求节点还是请求一个功能，作业都会失败，因为被请求的节点不能相互通信。我们建议将节点放置在不相干的分区中的独立网段。

对于具有蜻蜓网络的系统，用TopologyPlugin=topology/tree加上TopologyParam=dragonfly来配置Slurm。如果一个作业不能完全放在一个网络叶子开关内，作业将被分散到尽可能多的叶子开关中，以优化作业的网络带宽。

注意：当使用拓扑/树插件时，Slurm会识别为待处理作业提供最适合的网络交换机。如果节点有定义权重，这将覆盖基于网络拓扑结构的资源选择。如果通过节点权重优化资源选择比优化网络拓扑结构更重要，那么就不要使用拓扑/树形插件。

注意：Infiniband交换机的topology.conf文件可以使用这里的slurmibtopology工具自动生成。
https://ftp.fysik.dtu.dk/Slurm/slurmibtopology.sh。

注意：Omni-Path（OPA）交换机的topology.conf文件可以使用这里的opa2slurm工具自动生成。
https://gitlab.com/jtfrey/opa2slurm。

## 用户选项
为了与拓扑/树插件一起使用，用户还可以指定用于其工作的最大叶子开关数量，以及工作应该为该优化配置等待的最大时间。这个选项的语法是"--switches=count[@time]"。系统管理员可以使用带有max_switch_wait选项的SchedulerParameters配置参数来限制任何作业等待该优化配置的最大时间。

## 环境变量
如果使用拓扑/树插件，将设置两个环境变量来描述该作业的网络拓扑。注意，这些环境变量将包含每个节点上启动的任务的不同数据。这些环境变量的使用是由用户决定的。

SLURM_TOPOLOGY_ADDR：该值将被设置为可能参与任务通信的网络交换机名称，从系统的顶层交换机到叶层交换机，以节点名称结束。每一个硬件组件的名称之间用句号来分隔。

SLURM_TOPOLOGY_ADDR_PATTERN：只有在系统配置了拓扑/树形插件的情况下才会设置。该值将设置SLURM_TOPOLOGY_ADDR中列出的组件类型。每个组件将被识别为 "交换机 "或 "节点"。用一个句号来分隔每个硬件组件类型。