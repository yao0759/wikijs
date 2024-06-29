---
title: slurm--核心专用化
description: 翻译自：https://github.com/SchedMD/slurm/blob/master/doc/html/core_spec.shtml
published: true
date: 2024-06-29T05:39:22.020Z
tags: slurm
editor: markdown
dateCreated: 2024-06-29T05:39:22.020Z
---

# Core Specialization
核心专用化是一种功能，旨在将系统开销（系统中断等）隔离到计算节点上的指定核心。这可以减少应用程序中的上下文切换，从而缩短完成时间。作业进程将无法直接使用专用核心。

## Command Options

所有作业分配命令（**salloc**、**sbatch** 和 **srun**）都接受**-S** 或 **--core-spec**选项以及一个核心数值参数（如 **"-S 1"** 或 **"--core-spec=2"**）。核心数确定了在每个分配的计算节点上为系统开销预留的核心数。请注意，如果 slurm.conf 中未启用**AllowSpecResourcesUsage**，**--core-spec** 选项将被忽略。可以使用**scontrol**、**sview**或**squeue**命令查看每个作业的核心专业化计数。作业步骤的核心专业化计数指定将被忽略（例如，使用**salloc**或**sbatch**命令创建的作业分配中的**srun**命令）。使用带有**"%X "**格式选项的**squeue**命令可查看计数（默认输出格式不报告计数）。还可以使用 **scontrol** 和 **sview** 命令修改待处理任务的计数。

显式设置作业的专用核心值会隐式设置其**--exclusive**选项，为作业保留整个节点。作业将占用节点上的所有非专用 CPU，**scontrol**、**sview** 和 **squeue** 命令报告的作业 NumCPUs 值将反映所有分配节点上的所有非专用 CPUS，作业的账目也是如此。

请注意，由于隐式 **--exclusive**，如果请求的专用核心/线程数低于已分配节点的 **CoreSpecCount** 或 **CpuSpecList** 中的核心数，那么该步骤将可以访问所有非专用核心以及为该作业释放的专用核心。

例如，假设一个节点在 slurm.conf 中配置了 **AllowSpecResourcesUsage=yes**和 **CoreSpecCount=2**，该节点共有 16 个核心。如果作业指定**--core-spec=1**，隐式**--exclusive**将导致节点的独占分配，留下 15 个核心供作业使用，保留 1 个核心供系统使用。

在**sacct**中，步骤的已分配 CPU 将包括其可访问的专用核心或线程。但是，作业的已分配 CPU 数量从不包括专用核心或线程，以确保利用率报告的准确性。

下面是一个配置示例，将核心 0 和核心 1 设置为专用：

```
AllowSpecResourcesUsage=yes
Nodename=n0 Port=10100 CoresPerSocket=16 ThreadsPerCore=1 CpuSpecList=0-1
```

提交一个任务，要求将核心规格数设为 1（将 核心 1释放出来供工作使用）。

```
$ salloc --core-spec=1
salloc: Granted job allocation 4152
$ srun bash -c 'cat /proc/self/status |grep Cpus_'
Cpus_allowed:        fffe
Cpus_allowed_list:   1-15
```

注意作业 CPU 数量与步骤 CPU 数量的对比。

```
$ sacct -j 4152 -ojobid%20,alloccpus
               JobID  AllocCPUS

-------------------- ----------
                4152         14
    4152.interactive         15
              4152.0         15
```

## Core Selection

可使用 slurm.conf 文件中与每个节点相关联的 **CPUSpecList** 配置参数来确定用于特殊化的特定资源。如果配置了 **CoreSpecCount**，但未配置 **CPUSpecList**，则选择用于特殊化的核心将遵循下述分配算法。选择的第一个核心将是编号最高的套接字上编号最高的核心。随后选择的核心将是较低编号插座上编号最高的核心。如果需要额外的核心，则从每个插槽上编号次高的核心中选择。举例来说，一个节点有两个插槽，每个插槽有四个核心。将按以下顺序选择专用核心：

1. socket: 1 core: 3

2. socket: 0 core: 3
3. socket: 1 core: 2
4. socket: 0 core: 2
5. socket: 1 core: 1
6. socket: 0 core: 1
7. socket: 1 core: 0
8. socket: 0 core: 0

通过配置 **SchedulerParameters=spec_cores_first**，可以将 Slurm 配置为专门化第一个而非最后一个核心。在这种情况下，选择的第一个核心将是编号最低的插座上编号最低的核心。随后选择的核心将是编号较高插座上编号最低的核心。如果需要额外的核心，则从每个插槽上编号次低的核心中选择。

请注意，核心专业化保留可能会影响某些作业分配请求选项的使用，尤其是 **--cores-per-socket**。

## System Configuration

核心专用需要 **SelectType=cons_tres** 和 **task/cgroup** 任务插件。专用资源应在 slurm.conf 的节点规范行中使用 **CoreSpecCount** 或 **CPUSpecList** 选项进行配置，以确定要预留的 CPU。**MemSpecLimit** 选项可用于保留内存。这些资源将使用 Linux cgroups 保留。如果用户需要不同数量的专用核心，则应使用**--core-spec**选项，如上所述。

在其他配置中，作业的核心专用选项将被静默清除。此外，必须配置每个计算节点的核心数，或将 CPU 数配置为节点的核心数。如果未配置核心数，而 CPUs 值配置为超线程数，那么系统使用的将是超线程而非核心。

如果要授予用户控制其工作专用核心数量的权利，则必须将配置参数 **AllowSpecResourcesUsage**设置为 **1**。