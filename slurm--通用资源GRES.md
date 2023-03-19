---
title: slurm--通用资源调度GRES
description: 
published: true
date: 2023-03-19T15:36:52.964Z
tags: slurm
editor: markdown
dateCreated: 2023-03-19T15:36:52.964Z
---



## 概述
Slurm支持定义和安排任意的通用资源（GRES）的能力。为特定的GRES类型启用了额外的内置功能，包括图形处理单元（GPU）、CUDA多进程服务（MPS）设备，以及通过可扩展的插件机制的Sharding。

## 配置
默认情况下，集群的配置中没有启用GRES。你必须在slurm.conf配置文件中明确指定要管理哪些GRES。有关的配置参数是GresTypes和Gres。

更多细节请参见slurm.conf手册中的GresTypes和Gres。

请注意，每个节点的GRES规范的工作方式与管理其他资源的方式相同。如果发现节点的资源比配置的少，将被置于DRAIN状态。

摘录自slurm.conf文件的一个例子。

```
# Configure four GPUs (with MPS), plus bandwidth
GresTypes=gpu,mps,bandwidth
NodeName=tux[0-7] Gres=gpu:tesla:2,gpu:kepler:2,mps:400,bandwidth:lustre:no_consume:4G
```

每个拥有通用资源的计算节点通常都包含一个gres.conf文件，描述节点上有哪些资源可用，它们的数量，相关的设备文件和应该使用这些资源的核心。

在有些情况下，你可能想在一个节点上定义一个通用资源而不指定该GRES的数量。例如，一个节点的文件系统类型不会随着作业在该节点上的运行而减少价值。你可以使用no_consume标志来允许用户请求一个GRES，而不需要定义数量，当它被请求时就会被使用。

要查看可用的gres.conf配置参数，请参见gres.conf man页。

## 运行工作
工作不会被分配任何通用资源，除非在工作提交时使用选项特别要求。

- --gres

  每个节点需要的通用资源

- --gpus

  每个作业需要的GPU

- --gpus-per-node

  每个节点需要的GPU。相当于GPU的--gres选项。

- --gpus-per-socket

  每个插座所需的GPU。要求作业指定一个任务插座。

- --gpus-per-task

  每个任务需要的GPU。要求作业指定一个任务数。

所有这些选项都被salloc、sbatch和srun命令所支持。注意，所有的--gpu*选项只被Slurm的select/cons_tres插件支持。当select/cons_tres插件没有被配置时，请求这些选项的作业将被拒绝。--gres选项需要一个参数，以name[:type:count]的形式指定需要哪些通用资源和多少资源，而所有的--gpu*选项需要一个[type]:count形式的参数。名称与GresTypes和Gres配置参数指定的名称相同。类型确定该通用资源的特定类型（例如，GPU的特定型号）。计数指定需要多少资源，默认值为1。 例如。
sbatch --gres=gpu:kepler:2 ....

有几个额外的资源需求规格是专门针对GPU的，关于这些选项的详细描述可以在作业提交命令的手册中找到。至于--gpu*选项，这些选项只被Slurm的select/cons_tres插件支持。

- --cpus-per-gpu

  每个GPU分配的CPU的数量。

- --gpu-bind

  定义任务如何被绑定到GPU上。

- --gpu-freq

  指定GPU频率和/或GPU内存频率。

- --mem-per-gpu

  每个GPU分配的内存。

任务将根据需要被分配特定的通用资源以满足请求。如果作业被暂停，这些资源就不会被其他作业使用。

工作步骤可以从那些分配给工作的资源中分配通用资源，如上所述，使用srun命令的--gres选项。默认情况下，工作步骤将被分配所有分配给该工作的通用资源。如果需要，作业步骤可以明确指定与作业不同的通用资源数量。这个设计选择是基于一个场景，即每个作业执行了许多作业步骤。如果工作步骤默认被授予对所有通用资源的访问权，那么一些工作步骤就需要明确指定零的通用资源数量，我们认为这比较混乱。工作步骤可以被分配特定的通用资源，这些资源将不会被其他工作步骤使用。下面是一个简单的例子。

```
#!/bin/bash
#
# gres_test.bash
# Submit as follows:
# sbatch --gres=gpu:4 -n4 -N1-1 gres_test.bash
#
srun --gres=gpu:2 -n2 --exclusive show_device.sh &
srun --gres=gpu:1 -n1 --exclusive show_device.sh &
srun --gres=gpu:1 -n1 --exclusive show_device.sh &
wait
```

## 自动检测
如果在gres.conf中设置了AutoDetect=nvml或AutoDetect=rsmi，并且在节点上安装了相应的GPU管理库，并在Slurm配置过程中发现了它们，那么配置细节将自动填入任何系统检测到的GPU。这消除了在gres.conf中明确配置GPU的需要，尽管slurm.conf中的Gres=行仍然是需要的，以便告诉slurmctld期望有多少GRES。

默认情况下，所有系统检测到的设备都被添加到节点中。然而，如果gres.conf中的Type和File与系统中的一个GPU相匹配，任何其他明确指定的属性（例如Cores或Links）都可以与之进行重复检查。如果系统检测到的GPU与其匹配的GPU配置不同，那么该GPU将从节点中被省略，并出现错误。这允许gres.conf作为一个可选的理智检查，并通知管理员GPU属性中的任何意外变化。

如果不是所有系统检测到的设备都被slurm.conf配置所指定，那么相关的slurmd将被耗尽。然而，如果在gres.conf中手动指定这些设备（禁用AutoDetect），仍有可能使用系统中发现的设备的子集。

gres.conf文件的例子。

```
# Configure four GPUs (with MPS), plus bandwidth
AutoDetect=nvml
Name=gpu Type=gp100  File=/dev/nvidia0 Cores=0,1
Name=gpu Type=gp100  File=/dev/nvidia1 Cores=0,1
Name=gpu Type=p6000  File=/dev/nvidia2 Cores=2,3
Name=gpu Type=p6000  File=/dev/nvidia3 Cores=2,3
Name=mps Count=200  File=/dev/nvidia0
Name=mps Count=200  File=/dev/nvidia1
Name=mps Count=100  File=/dev/nvidia2
Name=mps Count=100  File=/dev/nvidia3
Name=bandwidth Type=lustre Count=4G Flags=CountOnly
```

在这个例子中，由于指定了AutoDetect=nvml，每个GPU的核心将根据系统上发现的与指定的类型和文件相匹配的GPU进行检查。由于没有指定Links，它将根据系统上发现的内容自动填写。如果没有找到匹配的系统GPU，则不会进行验证，GPU将被假定为配置中所说的那样。

要使Type与系统检测到的设备相匹配，它必须与slurmd通过AutoDetect机制报告的GPU名称完全匹配或成为其子串。这个GPU名称将用下划线替换所有空格。要查看GPU名称，请在slurm.conf中设置SlurmdDebug=debug2，然后重启或重新配置slurmd并查看slurmd日志。例如，用AutoDetect=nvml

```
debug:  gpu/nvml: init: init: GPU NVML plugin loaded
debug2: gpu/nvml: _nvml_init: Successfully initialized NVML
debug:  gpu/nvml: _get_system_gpu_list_nvml: Systems Graphics Driver Version: 450.36.06
debug:  gpu/nvml: _get_system_gpu_list_nvml: NVML Library Version: 11.450.36.06
debug2: gpu/nvml: _get_system_gpu_list_nvml: Total CPU count: 6
debug2: gpu/nvml: _get_system_gpu_list_nvml: Device count: 1
debug2: gpu/nvml: _get_system_gpu_list_nvml: GPU index 0:
debug2: gpu/nvml: _get_system_gpu_list_nvml:     Name: geforce_rtx_2060
debug2: gpu/nvml: _get_system_gpu_list_nvml:     UUID: GPU-g44ef22a-d954-c552-b5c4-7371354534b2
debug2: gpu/nvml: _get_system_gpu_list_nvml:     PCI Domain/Bus/Device: 0:1:0
debug2: gpu/nvml: _get_system_gpu_list_nvml:     PCI Bus ID: 00000000:01:00.0
debug2: gpu/nvml: _get_system_gpu_list_nvml:     NVLinks: -1
debug2: gpu/nvml: _get_system_gpu_list_nvml:     Device File (minor number): /dev/nvidia0
debug2: gpu/nvml: _get_system_gpu_list_nvml:     CPU Affinity Range - Machine: 0-5
debug2: gpu/nvml: _get_system_gpu_list_nvml:     Core Affinity Range - Abstract: 0-5
```

在这个例子中，GPU的名字被报告为geforce_rtx_2060。因此，在你的slurm.conf和gres.conf中，GPU类型可以设置为geforce、rtx、2060、geforce_rtx_2060，或者任何其他子串，slurmd应该能够将其与系统检测到的设备geforce_rtx_2060相匹配。

## GPU管理
在Slurm的GPU的GRES插件中，环境变量CUDA_VISIBLE_DEVICES被设置为每个作业步骤，以确定每个节点上有哪些GPU可供其使用。这个环境变量只有在任务在特定的计算节点上启动时才会被设置（salloc命令没有设置全局环境变量，而为sbatch命令设置的环境变量只反映在该节点上分配给该作业的GPU，即分配的零号节点）。CUDA 3.1版本（或更高版本）使用这个环境变量是为了在一个有GPU的节点上运行多个作业或作业步骤，并确保分配给每个作业的资源是唯一的。在上面的例子中，被分配的节点可能有四个或更多的图形设备。在这种情况下，CUDA_VISIBLE_DEVICES将为每个文件引用唯一的设备，输出结果可能类似于这样。

```
JobStep=1234.0 CUDA_VISIBLE_DEVICES=0,1
JobStep=1234.1 CUDA_VISIBLE_DEVICES=2
JobStep=1234.2 CUDA_VISIBLE_DEVICES=3
```

注意：请确保在gres.conf文件中指定文件参数，并确保它们是按数字递增的顺序排列。

CUDA_VISIBLE_DEVICES环境变量也将被设置在作业的Prolog和Epilog程序中。注意，如果Slurm被配置为使用Linux cgroup限制作业可见的设备文件，那么为作业设置的环境变量可能与为Prolog和Epilog设置的环境变量不同。这是因为Prolog和Epilog程序在任何Linux cgroup之外运行，而作业在cgroup内运行，因此可能有一组不同的可见设备。例如，如果一个作业被分配到设备"/dev/nvidia1"，那么CUDA_VISIBLE_DEVICES在Prolog和Epilog中会被设置为 "1"，而作业的CUDA_VISIBLE_DEVICES的值会被设置为 "0"（即作业可见的第一个GPU设备）。更多信息请参见《Prolog和Epilog指南》。

在可能的情况下，Slurm使用NVML自动确定系统中的GPU。NVML（它为nvidia-smi工具提供动力）按照PCI总线ID对GPU进行编号。为了使这个编号与CUDA报告的编号一致，CUDA_DEVICE_ORDER环境变量必须被设置为CUDA_DEVICE_ORDER=PCI_BUS_ID。

GPU设备文件（例如/dev/nvidia1）是基于Linux的次要号码分配，而NVML的设备号码是通过PCI总线ID分配的，从低到高。这两者之间的映射是不确定的，并且依赖于系统，在硬件或操作系统改变后，在启动时可能会有所不同。在大多数情况下，这种分配似乎相当稳定。然而，需要在启动后进行检查，以保证一个GPU设备被分配到一个特定的设备文件。

关于CUDA_VISIBLE_DEVICES和CUDA_DEVICE_ORDER环境变量的更多信息，请查阅NVIDIA CUDA文档。

## MPS管理
CUDA多进程服务（MPS）提供了一种机制，GPU可以被多个作业共享，每个作业被分配一定比例的GPU的资源。一个节点上可用的MPS资源总数应该在slurm.conf文件中进行配置（例如 "NodeName=tux[1-16] Gres=gpu:2,mps:200"）。在gres.conf文件中，有几个选项可以用来配置MPS，下面是一些例子。

1. 没有MPS配置。slurm.conf中定义的gres/mps元素的数量将均匀地分布在节点上配置的所有GPU上。例如，"NodeName=tux[1-16] Gres=gpu:2,mps:200 "将在两个GPU上各配置100个gres/mps资源数。

2. MPS配置只包括名称和计数参数。gres/mps元素的数量将均匀地分布在节点上配置的所有GPU上。这与情况1类似，但在gres.conf文件中放置了重复的配置。
3. MPS的配置包括名称、文件和计数参数。每个File参数应该确定一个GPU的设备文件路径，Count应该确定该特定GPU设备可用的gres/mps资源数量。这在异质环境中可能是有用的。例如，一个节点上的一些GPU可能比其他的更强大，因此与更高的gres/mps计数有关。另一个用例是防止一些GPU被用于MPS（即它们的MPS计数为零）。

请注意，gres/mps的Type和Cores参数被忽略。这些信息是从gres/gpu配置中复制过来的。

注意Count参数被转换为百分比，因此该值通常是100的倍数。

请注意，如果安装了NVIDIA的NVML库，GPU配置（即类型、文件、核心和链接数据）将自动从该库中收集，无需在gres.conf文件中记录。

请注意，同一个GPU既可以作为GRES的GPU类型分配，也可以作为GRES的MPS类型分配，但不能同时分配。换句话说，一旦一个GPU被分配为gres/gpu资源，它就不能作为gres/mps使用。同样地，一旦一个GPU被分配为gres/mps资源，它将不能作为gres/gpu使用。然而，同一个GPU可以作为MPS通用资源分配给属于多个用户的多个作业，只要分配给作业的MPS总数量不超过配置的数量。另外，由于共享GRES（MPS）不能与共享GRES（GPU）同时分配，这个选项只分配所有的共享GRES，没有底层的共享GRES。下面是Slurm的gres.conf文件的一些配置例子。

```
# Example 1 of gres.conf
# Configure four GPUs (with MPS)
AutoDetect=nvml
Name=gpu Type=gp100 File=/dev/nvidia0 Cores=0,1
Name=gpu Type=gp100 File=/dev/nvidia1 Cores=0,1
Name=gpu Type=p6000 File=/dev/nvidia2 Cores=2,3
Name=gpu Type=p6000 File=/dev/nvidia3 Cores=2,3
# Set gres/mps Count value to 100 on each of the 4 available GPUs
Name=mps Count=400
 # Example 2 of gres.conf
# Configure four different GPU types (with MPS)
AutoDetect=nvml
Name=gpu Type=gtx1080 File=/dev/nvidia0 Cores=0,1
Name=gpu Type=gtx1070 File=/dev/nvidia1 Cores=0,1
Name=gpu Type=gtx1060 File=/dev/nvidia2 Cores=2,3
Name=gpu Type=gtx1050 File=/dev/nvidia3 Cores=2,3
Name=mps Count=1300   File=/dev/nvidia0
Name=mps Count=1200   File=/dev/nvidia1
Name=mps Count=1100   File=/dev/nvidia2
Name=mps Count=1000   File=/dev/nvidia3
```

注意：gres/mps需要使用select/cons_tres插件。

对MPS的作业请求的处理与其他GRES相同，除了请求必须在每个节点上只使用一个GPU来满足，并且每个节点只能配置一个GPU来使用MPS。例如，"--gres=mps:50 "的作业请求不会通过使用一个节点上的20%的GPU和30%的第二个GPU来满足。来自不同用户的多个作业可以同时在一个节点上使用MPS。请注意，GRES类型的GPU和MPS不能在一个作业中被请求。另外，请求MPS资源的作业不能指定GPU频率。

一个prolog程序应该被用来根据需要启动和停止MPS服务器。在Slurm的发行版中，有一个prolog脚本的样本，位于etc/prolog.example。它的操作模式是，如果一个作业被分配了gres/mps资源，那么Prolog将设置CUDA_VISIBLE_DEVICES, CUDA_MPS_ACTIVE_THREAD_PERCENTAGE和SLURM_JOB_UID环境变量。然后Prolog应该确保为该GPU和用户启动一个MPS服务器（UID == User ID）。它也会在本地文件中记录GPU的设备ID。如果一个作业被分配了gres/gpu资源，那么Prolog将设置CUDA_VISIBLE_DEVICES和SLURM_JOB_UID环境变量（没有CUDA_MPS_ACTIVE_THREAD_PERCENTAGE）。然后Prolog应该终止与该GPU相关的任何MPS服务器。可能有必要根据本地环境的需要来修改这个脚本。欲了解更多信息，请参阅Prolog和Epilog指南。

请求MPS资源的工作将设置CUDA_VISIBLE_DEVICES和CUDA_DEVICE_ORDER环境变量。设备ID是相对于那些在MPS服务器控制下的资源而言的，在目前的实现中，设备ID的值总是为0（每个节点只有一个GPU可用于MPS模式）。作业也将有CUDA_MPS_ACTIVE_THREAD_PERCENTAGE环境变量，设置为该作业在指定GPU上可用的MPS资源的百分比。这个百分比将根据Gres上配置的Count的部分来计算，并分配给一个作业的步骤。例如，一个请求"--gres=gpu:200 "并使用上述配置例子2的作业将被分配到

15% of the gtx1080 (File=/dev/nvidia0, 200 x 100 / 1300 = 15), or

16% of the gtx1070 (File=/dev/nvidia0, 200 x 100 / 1200 = 16), or

18% of the gtx1060 (File=/dev/nvidia0, 200 x 100 / 1100 = 18), or

20% of the gtx1050 (File=/dev/nvidia0, 200 x 100 / 1000 = 20).

另一种操作模式是允许作业被分配到整个GPU，然后根据作业中的注释来触发MPS服务器的启动。例如，如果一个作业被分配到整个GPU，那么在作业中搜索 "mps-per-gpu "或 "mps-per-node "的注释（使用 "scontrol show job "命令），并将其作为基础，分别在每个GPU或所有GPU上启动一个MPS守护程序。

关于MPS的更多信息，请查阅NVIDIA多进程服务文档。

请注意，在以前版本的NVIDIA驱动程序中存在一个漏洞，在共享GPU时可能会影响用户。更多信息可以在CVE-2018-6260和安全公告中找到。NVIDIA GPU显示驱动程序 - 2019年2月。

关于不同用户之间的GPU共享，NVIDIA MPS有一个内置限制。一个系统中只有一个用户可以拥有一个活跃的MPS服务器，而MPS控制守护进程将排队等待来自不同用户的MPS服务器激活请求，从而导致用户之间对GPU的序列化独占访问（参见MPS文档中的第2.3.1.1节 - 限制）。因此，不同的用户不能真正在MPS的GPU上同时运行；相反，GPU将在用户之间进行时间分割（关于这个过程的图示，请参见MPS文档中的第3.3节 - 配置顺序）。

## MIG管理
从21.08版本开始，Slurm现在支持NVIDIA多实例GPU（MIG）设备。这项功能允许一些较新的NVIDIA GPU（如A100）将一个GPU分割成最多七个独立的、隔离的GPU实例。Slurm可以将这些MIG实例视为独立的GPU，并完成c组隔离和任务绑定。

要在Slurm中配置MIG，在gres.conf中为有MIG的节点指定AutoDetect=nvml，并在slurm.conf中指定Gres，就像MIG是普通GPU一样。可以指定一个可选的GRES类型，以区分不同大小的MIG，以及集群中的其他GPU。该类型必须是 "MIG Profile "字符串的子串，该字符串由节点在其slurmd日志中的debug2日志级别下报告。其他的MIG属性也将被显示出来，包括MIG UUID、GPU实例（GI）ID、计算实例（CI）ID、GI和CI次要号码、相关的MIG设备文件和 "UniqueID"（Slurm将通过CUDA_VISIBLE_DEVICES来选择MIG的值）。

MultipleFiles参数允许你为GPU卡指定多个设备文件。

MIG不支持理智检查的自动检测模式。Slurm希望MIG设备已经被分区，不支持MIG的动态分区。

关于NVIDIA MIG的更多信息（包括如何对其进行分区），请参见MIG用户指南。

## 分片
分片提供了一种通用机制，可以让多个作业共享GPU。虽然它确实允许多个作业在一个给定的GPU上运行，但它并不限制在GPU上运行的进程，它只允许GPU被共享。因此，分片技术在同质化工作流程中效果最好。建议限制一个节点上的分片数量，使其等于该节点上可同时运行的最大作业数（即核心数）。一个节点上可用的分片总数应该在slurm.conf文件中配置（例如，"NodeName=tux[1-16] Gres=gpu:2,shard:64"）。在gres.conf文件中，有几个选项可以用来配置分片，下面列出了一些例子。

1. 没有配置分片。slurm.conf中定义的gres/shard元素的数量将均匀地分布在节点上配置的所有GPU上。例如，"NodeName=tux[1-16] Gres=gpu:2,shard:64 "将在两个GPU上分别配置32个gres/shard资源。
2. 分片配置只包括名称和计数参数。gres/shard元素的数量将均匀地分布在节点上配置的所有GPU上。这与情况1类似，但在gres.conf文件中放置了重复的配置。
3. Shard配置包括Name、File和Count参数。每个File参数应该确定一个GPU的设备文件路径，Count应该确定该特定GPU设备可用的gres/shard资源的数量。这在异质环境中可能是有用的。例如，一个节点上的一些GPU可能比其他的更强大，因此与更高的gres/shard计数有关。另一个用例是防止一些GPU被用于分片（即它们的分片计数为零）。

注意，gres/shard的Type和Cores参数被忽略了。这些信息是从gres/gpu配置中复制过来的。

请注意，如果安装了NVIDIA的NVML库，GPU配置（即类型、文件、核心和链接数据）将自动从该库中收集，无需在gres.conf文件中记录。

注意同一个GPU既可以作为GRES的GPU类型分配，也可以作为GRES的分片类型分配，但不能同时分配。换句话说，一旦一个GPU被分配为gres/gpu资源，它将不能作为gres/shard使用。同样地，一旦一个GPU被分配为gres/shard资源，它将不能作为gres/gpu使用。然而，同一个GPU可以作为shard通用资源分配给属于多个用户的多个作业，只要分配给作业的SHARD总数量不超过配置的数量。

为了正确配置，适当的节点需要将shard关键字添加为相关节点的GRES，并添加到GresTypes参数中。如果你想在会计中跟踪碎片，那么碎片也需要被添加到AccountingStorageTRES中。参见slurm.conf例子中的相关设置。

```
AccountingStorageTRES=gres/gpu,gres/shard
GresTypes=gpu,shard
NodeName=tux[1-16] Gres=gpu:2,shard:64
```

Slurm的gres.conf文件的一些配置例子如下所示

```
# Example 1 of gres.conf
# Configure four GPUs (with Sharding)
AutoDetect=nvml
Name=gpu Type=gp100 File=/dev/nvidia0 Cores=0,1
Name=gpu Type=gp100 File=/dev/nvidia1 Cores=0,1
Name=gpu Type=p6000 File=/dev/nvidia2 Cores=2,3
Name=gpu Type=p6000 File=/dev/nvidia3 Cores=2,3
# Set gres/shard Count value to 8 on each of the 4 available GPUs
Name=shard Count=32
 # Example 2 of gres.conf
# Configure four different GPU types (with Sharding)
AutoDetect=nvml
Name=gpu Type=gtx1080 File=/dev/nvidia0 Cores=0,1
Name=gpu Type=gtx1070 File=/dev/nvidia1 Cores=0,1
Name=gpu Type=gtx1060 File=/dev/nvidia2 Cores=2,3
Name=gpu Type=gtx1050 File=/dev/nvidia3 Cores=2,3
Name=shard Count=8    File=/dev/nvidia0
Name=shard Count=8    File=/dev/nvidia1
Name=shard Count=8    File=/dev/nvidia2
Name=shard Count=8    File=/dev/nvidia3
```

注意：gres/shard需要使用select/cons_tres插件。

对shard的作业请求不能指定GPU频率。

请求shards资源的作业将有CUDA_VISIBLE_DEVICES、ROCR_VISIBLE_DEVICES或GPU_DEVICE_ORDINAL环境变量的设置，这将与GPU的情况相同。