---
title: slurm--CPU管理用户和管理员指南
description: slurm中文翻译系列，机翻后纠正了一点，发现其他错误望指出，来源：https://github.com/SchedMD/slurm/blob/master/doc/html/cpu_management.shtml
published: false
date: 2023-04-16T14:37:11.685Z
tags: slurm
editor: markdown
dateCreated: 2023-04-16T14:37:11.685Z
---

## 概述

本指南的目的是帮助Slurm用户和管理员选择配置选项和组成命令行来管理作业、步骤和任务对CPU资源的使用。该文件分为以下几个部分。

- 概述

- 由Slurm执行的CPU管理步骤
- 获取作业/步骤/任务的CPU使用信息
- CPU管理和Slurm会计
- CPU管理实例

通过用户命令进行的CPU管理受制于Slurm管理员选择的配置参数。不同的CPU管理选项之间的相互作用是复杂的，往往难以预测。可能需要进行一些实验，以发现产生所需结果所需的选项的确切组合。用户和管理员应该参考`slurm.conf`、`cgroup.conf`、`salloc`、`sbatch`和`srun`的手册，以了解每个选项的详细解释。下面的html文件也可能是有用的：

本文档仅描述了常规Linux集群的Slurm CPU管理。关于Cray ALPS系统的信息，请参考相应的文件。

## 由Slurm执行的CPU管理步骤

Slurm使用四个基本步骤来管理一个作业/步骤的CPU资源。

- 第1步：选择节点
- 第2步：从选定的节点分配CPU
- 第3步：将任务分配到选定的节点上
- 第4步：可选的分配和绑定任务到节点内的CPU上

### 第1步：选择节点

在步骤1中，Slurm选择CPU资源分配给作业或作业步骤的节点集。因此，节点选择受到许多控制CPU分配的配置和命令行选项的影响（下面的步骤2）。如果`SelectType=select/linear`被配置，所选节点的所有资源将被分配给作业/步骤。如果`SelectType`被配置为`select/cons_res`或`select/cons_tres`，单个套接字、内核和线程可以作为消耗性资源从所选节点中分配。可消耗资源的类型由`SelectTypeParameters`定义。

步骤1是由slurmctld和select插件执行的。

**控制步骤1的slurm.conf选项**

| slurm.conf参数       | 值                                                           | 描述                                                         |
| -------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| NodeName             | <节点的名称> 加上额外的参数。详见man页。                     | 定义了一个节点。这包括节点上的板卡、插座、核心、线程和处理器（逻辑CPU）的数量和布局。 |
| PartitionName        | <分区的名称> 加上额外的参数。详见手册页。                    | 定义了一个分区。分区定义的几个参数会影响节点的选择（例如，Nodes, OverSubscribe, MaxNodes）。 |
| SlurmdParameters     | config_overrides                                             | 控制节点定义中的信息如何被使用。                             |
| SelectType           | select/linear ; select/linear ; select/cons_tres             | 控制CPU资源是以整个节点为单位分配给作业和作业步骤，还是以消耗性资源（套接字、内核或线程）为单位分配。 |
| SelectTypeParameters | CR_CPU;CR_CPU_Memory;CR_Core;CR_Core_Memory;CR_Socket;CR_Socket_Memory加上额外的选项。详见man页。 | 定义可消耗的资源类型，并控制选择插件的CPU资源分配的其他方面。 |

**控制步骤1的srun/salloc/sbatch命令行选项**

| 命令行选项            | 值                                               | 描述                                                         |
| --------------------- | ------------------------------------------------ | ------------------------------------------------------------ |
| -B, --extra-node-info | `<sockets[:cores[:threads]]>`                    | 将节点选择限制在具有指定插座、内核和线程布局的节点上。       |
| -C, --constraint      | `<list>`                                         | 将节点选择限制在具有指定属性的节点上                         |
| --contiguous          | `N/A`                                            | 将节点选择限制在相邻的节点上                                 |
| --cores-per-socket    | `<cores>`                                        | 将节点选择限制在每个插座至少有指定核心数的节点上。           |
| -c, --cpus-per-task   | `<ncpus>`                                        | 控制每个任务分配的CPU数量                                    |
| --exclusive           | `N/A`                                            | 防止与其他作业共享分配的节点。将CPU分配给作业步骤。          |
| -F, --nodefile        | `<node file>`                                    | 包含要为作业选择的特定节点列表的文件（仅适用于salloc和sbatch）。 |
| --hint                | `compute_bound ; memory_bound ; [no]multithread` | 对CPU资源分配的额外控制                                      |
| --mincpus             | `<n>`                                            | 控制每个节点分配的最小CPU数量                                |
| -N, --nodes           | `<minnodes[-maxnodes]>`                          | 控制分配给工作的最小/最大的节点数                            |
| -n, --ntasks          | `<number>`                                       | 控制要为工作创建的任务数量。                                 |
| --ntasks-per-core     | `<number>`                                       | 控制每个分配的核心的最大任务数                               |
| --ntasks-per-socket   | `<number>`                                       | 控制每个分配的socket的最大任务数                             |
| --ntasks-per-node     | `<number>`                                       | 控制每个分配节点的最大任务数                                 |
| -O, --overcommit      | `N/A`                                            | 允许分配的CPU数量少于任务的数量                              |
| -p, --partition       | `<partition_names>`                              | 控制哪个分区被用于工作                                       |
| -s, --oversubscribe   | `N/A`                                            | 允许与其他工作共享分配的节点                                 |
| --sockets-per-node    | `<sockets>`                                      | 将节点选择限制在至少有指定数量套接字的节点上。               |
| --threads-per-core    | `<threads>`                                      | 将节点选择限制在每核至少有指定线程数的节点上。               |
| -w, --nodelist        | `<host1,host2,... or filename>`                  | 将被分配给工作的特定节点的列表                               |
| -x, --exclude         | `<host1,host2,... or filename>`                  | 将被排除在作业分配之外的特定节点列表                         |
| -Z, --no-allocate     | `N/A`                                            | 绕过正常分配（仅对用户 "SlurmUser "和 "root "可用的特权选项）。 |

### 第2步：从选定的节点中分配CPU

在步骤2中，Slurm从步骤1中选择的节点集合中为一个作业/步骤分配CPU资源。因此，CPU分配受到与节点选择有关的配置和命令行选项的影响。如果`SelectType=select/linear`被配置，所选节点上的所有资源将被分配给作业/步骤。如果`SelectType`被配置为`select/cons_res`或`select/cons_tres`，单个套接字、内核和线程可以作为消耗性资源从所选节点中分配。可消耗资源的类型由`SelectTypeParameters`定义。

当使用`select/cons_res`或`select/cons_tres`的`SelectType`时，默认的跨节点分配方法是块状分配（在使用另一个节点之前分配一个节点的所有可用CPU）。节点内的默认分配方法是循环分配（在一个节点内的套接字之间以轮流方式分配可用的CPU）。用户可以使用下面描述的适当的命令行选项来覆盖默认行为。分配方法的选择可能会影响哪些特定的CPU被分配给作业/步骤。

**第2步由slurmctld和选择插件执行。**

| slurm.conf参数       | 值                                                           | 描述                                                         |
| -------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| NodeName             | <节点的名称> 加上额外的参数。详见man页。                     | 定义了一个节点。这包括节点上的板卡、插座、核心、线程和处理器（逻辑CPU）的数量和布局。 |
| PartitionName        | <分区的名称> 加上额外的参数。详见手册页。                    | 定义了一个分区。分区定义的几个参数会影响节点的选择（例如，Nodes, OverSubscribe, MaxNodes）。 |
| SlurmdParameters     | config_overrides                                             | 控制节点定义中的信息如何被使用。                             |
| SelectType           | select/linear ; select/linear ; select/cons_tres             | 控制CPU资源是以整个节点为单位分配给作业和作业步骤，还是以消耗性资源（套接字、内核或线程）为单位分配。 |
| SelectTypeParameters | CR_CPU;CR_CPU_Memory;CR_Core;CR_Core_Memory;CR_Socket;CR_Socket_Memory加上额外的选项。详见man页。 | 定义可消耗的资源类型，并控制选择插件的CPU资源分配的其他方面。 |

**控制步骤2的srun/salloc/sbatch命令行选项**

| 命令行选项            | 值                                               | 描述                                                         |
| --------------------- | ------------------------------------------------ | ------------------------------------------------------------ |
| -B, --extra-node-info | `<sockets[:cores[:threads]]>`                    | 将节点选择限制在具有指定插座、内核和线程布局的节点上。       |
| -C, --constraint      | `<list>`                                         | 将节点选择限制在具有指定属性的节点上                         |
| --contiguous          | `N/A`                                            | 将节点选择限制在相邻的节点上                                 |
| --cores-per-socket    | `<cores>`                                        | 将节点选择限制在每个插座至少有指定核心数的节点上。           |
| -c, --cpus-per-task   | `<ncpus>`                                        | 控制每个任务分配的CPU数量                                    |
| --exclusive           | `N/A`                                            | 防止与其他作业共享分配的节点。将CPU分配给作业步骤。          |
| -F, --nodefile        | `<node file>`                                    | 包含要为作业选择的特定节点列表的文件（仅适用于salloc和sbatch）。 |
| --hint                | `compute_bound ; memory_bound ; [no]multithread` | 对CPU资源分配的额外控制                                      |
| --mincpus             | `<n>`                                            | 控制每个节点分配的最小CPU数量                                |
| -N, --nodes           | `<minnodes[-maxnodes]>`                          | 控制分配给工作的最小/最大的节点数                            |
| -n, --ntasks          | `<number>`                                       | 控制要为工作创建的任务数量。                                 |
| --ntasks-per-core     | `<number>`                                       | 控制每个分配的核心的最大任务数                               |
| --ntasks-per-socket   | `<number>`                                       | 控制每个分配的socket的最大任务数                             |
| --ntasks-per-node     | `<number>`                                       | 控制每个分配节点的最大任务数                                 |
| -O, --overcommit      | `N/A`                                            | 允许分配的CPU数量少于任务的数量                              |
| -p, --partition       | `<partition_names>`                              | 控制哪个分区被用于工作                                       |
| -s, --oversubscribe   | `N/A`                                            | 允许与其他工作共享分配的节点                                 |
| --sockets-per-node    | `<sockets>`                                      | 将节点选择限制在至少有指定数量套接字的节点上。               |
| --threads-per-core    | `<threads>`                                      | 将节点选择限制在每核至少有指定线程数的节点上。               |
| -w, --nodelist        | `<host1,host2,... or filename>`                  | 将被分配给工作的特定节点的列表                               |
| -x, --exclude         | `<host1,host2,... or filename>`                  | 将被排除在作业分配之外的特定节点列表                         |
| -Z, --no-allocate     | `N/A`                                            | 绕过正常分配（仅对用户 "SlurmUser "和 "root "可用的特权选项）。 |

### 第3步：将任务分配给选定的节点

在步骤3中，Slurm将任务分配给在步骤1中为工作/步骤选择的节点。每个任务只分配给一个节点，但每个节点可以分配一个以上的任务。除非为作业指定了CPU对任务的过度承诺，否则分配到一个节点上的任务数量受到该节点上分配的CPU数量和每个任务的CPU数量的限制。如果配置了消耗性资源，或者允许资源共享，来自多个作业/步骤的任务可以在同一个节点上并发运行。

第3步是由slurmctld执行的。

| slurm.conf参数      | 值                                                      | 描述                                                         |
| ------------------- | ------------------------------------------------------- | ------------------------------------------------------------ |
| MaxTasksPerNode     | `<number>`                                              | 控制一个工作步骤在单个节点上可以生成的最大任务数。           |
| --distribution, -m  | `block;cyclic;arbitrary;plane=<options>[:block;cyclic]` | 第一个指定的分布（在": "之前）控制任务被分配到每个选定节点的顺序。注意，这个选项不影响分配到每个节点的任务数量，只影响分配的顺序。 |
| --ntasks-per-core   | `<number>`                                              | 控制每个分配的核心的最大任务数                               |
| --ntasks-per-socket | `<number>`                                              | 控制每个分配的socket的最大任务数                             |
| --ntasks-per-node   | `<number>`                                              | 控制每个分配节点的最大任务数                                 |

### 步骤4：可选择的任务分配和绑定到节点内的CPU上
在可选的步骤4中，Slurm将每个任务分配并绑定到在步骤3中分配任务的节点上的一个指定的CPU子集。分发到同一节点的不同任务可能被绑定到相同的CPU子集或不同的子集上。这个步骤被称为任务亲和性或`task/CPU`绑定。

第4步是由slurmd和任务插件执行的。


| slurm.conf parameter | Possible values                      | Description                                                  |
| -------------------- | ------------------------------------ | ------------------------------------------------------------ |
| TaskPlugin           | task/none,task/affinity ,task/cgroup | Controls whether this step is enabled and which task plugin to use |



| cgroupslurm.conf parameter | Possible values | Description                                                  |
| -------------------------- | --------------- | ------------------------------------------------------------ |
| ConstrainCores             | yes,no          | Controls whether jobs are constrained to their allocated CPUs |



| Command line option | Possible values                                         | Description                                                  |
| ------------------- | ------------------------------------------------------- | ------------------------------------------------------------ |
| --cpu-bind          | See man page                                            | Controls binding of tasks to CPUs (srun only)                |
| --ntasks-per-core   | `<number>     `                                           | Controls the maximum number of tasks per allocated core      |
| --distribution, -m  | block,cyclic,arbitrary,plane=<options>,[:block\|cyclic] | The second specified distribution (after the ":") controls the sequence in which tasks are distributed to allocated CPUs within a node for binding of tasks to CPUs |

  
## 关于CPU管理步骤的补充说明

对于消耗性资源，用户必须了解cpu分配（步骤2）和任务亲和性/绑定（步骤4）之间的区别。作为消耗性资源的CPU的独占（非共享）分配限制了可以并发使用一个节点的作业/步骤/任务的数量。但它并不限制分配到该节点上的每个任务可以使用该节点上的CPU集合。除非使用某种形式的CPU/任务绑定（例如，任务或spank插件），所有分配到一个节点的任务可以使用节点上的所有CPU，包括没有分配给他们的作业/步骤的CPU。这可能会对性能产生意想不到的不利影响，因为它允许一个作业使用专门分配给另一个作业的CPU。出于这个原因，在不配置任务亲和性的情况下配置可消耗资源可能是不可取的。请注意，当配置了选择/线性（整个节点分配）时，任务亲和性也是有用的，通过将每个任务限制在一个节点上的特定插座或其他CPU资源子集，来提高性能。

## 获取作业/步骤/任务的CPU使用信息

没有简单的方法可以为一个作业/步骤（分配、分布和绑定）生成一套全面的CPU管理信息。然而，有几个命令/选项提供了有限的CPU使用信息：
  

| Command/Option                                               | Information                                                  |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| scontrol show job option: `--details`                        | This option provides a list of the nodes selected for the job and the CPU ids allocated to the job on each node. Note that the CPU ids reported by this command are Slurm abstract CPU ids, not Linux/hardware CPU ids (as reported by, for example, /proc/cpuinfo). |
| Linux command: `env`                                         | Man. Slurm environment variables provide information related to node and CPU usage: SLURM_JOB_CPUS_PER_NODE<br/>SLURM_CPUS_PER_TASK<br/>SLURM_CPU_BIND<br/>SLURM_DISTRIBUTION<br/>SLURM_JOB_NODELIST<br/>SLURM_TASKS_PER_NODE<br/>SLURM_STEP_NODELIST<br/>SLURM_STEP_NUM_NODES<br/>SLURM_STEP_NUM_TASKS<br/>SLURM_STEP_TASKS_PER_NODE<br/>SLURM_JOB_NUM_NODES<br/>SLURM_NTASKS<br/>SLURM_NPROCS<br/>SLURM_CPUS_ON_NODE<br/>SLURM_NODEID<br/>SLURMD_NODENAME |
| srun option: `--cpu-bind=verbose`                            | This option provides a list of the CPU masks used by task affinity to bind tasks to CPUs. Note that the CPU ids represented by these masks are Linux/hardware CPU ids, not Slurm abstract CPU ids as reported by scontrol, etc. |
| srun/salloc/sbatch option: `-l`                              | This option adds the task id as a prefix to each line of output from a task sent to stdout/stderr. This can be useful for distinguishing node-related and CPU-related information by task id for multi-task jobs/steps. |
| Linux command:`cat /proc/<pid>/status | grep Cpus_allowed_list` | Given a task's pid (or "self" if the command is executed by the task itself), this command produces a list of the CPU ids bound to the task. This is the same information that is provided by `--cpu-bind=verbose`, but in a more readable format. |

### 关于CPU编号的说明

Slurm知道的逻辑CPU的数量和布局在`slurm.conf`的节点定义中描述。这可能与实际硬件上的物理CPU布局不同。由于这个原因，Slurm产生了它自己的内部，或 `"abstract"`的CPU编号。这些数字可能与Linux已知的物理或 `"machine"`的CPU数字不一致。


## CPU管理和Slurm核算
Slurm用户的CPU管理受制于Slurm Accounting的限制。会计限制可以在用户、组和集群层面上应用于CPU的使用。详情请见`sacctmgr` man页。


## CPU管理实例
  
下面的例子说明了使用Slurm管理CPU资源的一些情况。许多其他的场景也是可能的。在每个例子中，都假设每个节点上的所有CPU都可以分配。

节点和分区配置示例：
- Example Node and Partition Configuration
- Example 1: Allocation of whole nodes
- Example 2: Simple allocation of cores as consumable resources
- Example 3: Consumable resources with balanced allocation across nodes
- Example 4: Consumable resources with minimization of resource fragmentation
- Example 5: Consumable resources with cyclic distribution of tasks to nodes
- Example 6: Consumable resources with default allocation and plane distribution of tasks to nodes
- Example 7: Consumable resources with overcommitment of CPUs to tasks
- Example 8: Consumable resources with resource sharing between jobs
- Example 9: Consumable resources on multithreaded node, allocating only one thread per core
- Example 10: Consumable resources with task affinity and core binding
- Example 11: Consumable resources with task affinity and socket binding, Case 1
- Example 12: Consumable resources with task affinity and socket binding, Case 2
- Example 13: Consumable resources with task affinity and socket binding, Case 3
- Example 14: Consumable resources with task affinity and customized allocation and distribution
- Example 15: Consumable resources with task affinity to optimize the performance of a multi-task, multi-thread job
- Example 16: Consumable resources with task cgroup
  
## 节点和分区配置示例

在这些例子中，Slurm集群包含以下节点：
 

| Nodename                          | n0   | n1   | n2   | n3   |
| --------------------------------- | ---- | ---- | ---- | ---- |
| Number of Sockets                 | 2    | 2    | 2    | 2    |
| Number of Cores per Socket        | 4    | 4    | 4    | 4    |
| Total Number of Cores             | 8    | 8    | 8    | 8    |
| Number of Threads (CPUs) per Core | 1    | 1    | 1    | 2    |
| Total Number of CPUs              | 8    | 8    | 8    | 16   |

以及以下分区：

| PartitionName | regnodes | hypernode |
| ------------- | -------- | --------- |
| Nodes         | n0 n1 n2 | n3        |
| Default       | YES      |           |

这些实体在slurm.conf中定义如下：

```
Nodename=n0 NodeAddr=node0 Sockets=2 CoresPerSocket=4 ThreadsPerCore=1 Procs=8
Nodename=n1 NodeAddr=node1 Sockets=2 CoresPerSocket=4 ThreadsPerCore=1 Procs=8 State=IDLE
Nodename=n2 NodeAddr=node2 Sockets=2 CoresPerSocket=4 ThreadsPerCore=1 Procs=8 State=IDLE
Nodename=n3 NodeAddr=node3 Sockets=2 CoresPerSocket=4 ThreadsPerCore=2 Procs=16 State=IDLE
PartitionName=regnodes Nodes=n0,n1,n2 OverSubscribe=YES Default=YES State=UP
PartitionName=hypernode Nodes=n3 State=UP
```

这些例子显示了对`cons_res`选择类型插件的使用，但他们可以使用`cons_tres`插件，效果也一样。


### 