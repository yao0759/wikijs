---
title: slurm--cgroup v2插件
description: slurm中文翻译系列，机翻后纠正了一点，发现其他错误望指出
published: true
date: 2023-03-19T15:28:23.247Z
tags: slurm, hpc
editor: markdown
dateCreated: 2022-09-18T10:35:45.968Z
---

Slurm为cgroup v2的系统提供支持。
这个cgroup版本的文档可以在kernel.org [Control Cgroup v2文档](https://www.kernel.org/doc/html/latest/admin-guide/cgroup-v2.html)中找到。

cgroup/v2插件是Slurm内部的API，被其他插件使用，如proctrack/cgroup、task/cgroup和jobacctgather/cgroup。本文档概述了它是如何设计的，目的是为了更好地了解当Slurm用这个插件约束资源时系统上发生了什么。

在阅读本文档之前，我们假设你已经阅读了[cgroup v2内核文档](https://www.freedesktop.org/wiki/Software/systemd/ControlGroupInterface/)，并且熟悉了大部分的概念和术语。阅读 systemd 的cgroup接口文档同样重要，因为 cgroup/v2 需要与 systemd 进行交互，很多概念会有重叠。最后，建议你了解 [eBPF 技术](https://ebpf.io/what-is-ebpf/)的概念，因为在 cgroup v2 中，设备 cgroup 控制器是基于 eBPF 的。

## 遵循cgroup v2规则

内核的cgroup v2有两个特殊性，影响Slurm需要如何构造其内部的cgroup树。

### 自上而下的约束

资源是自上而下分布到树上的，所以只有当父节点在其cgroup.controllers文件中列出并添加到其cgroup.subtree_control中时，一个controller才能在cgroup目录中使用。另外，如果一个或多个子节点启用了控制器，那么在子树上激活的controller不能被禁用。对于Slurm来说，这意味着我们需要通过修改cgroup.subtree_control来对我们的层次结构进行这种管理，并为子代启用所需的controller 。

### 没有内部进程约束

除了 root cgroup之外，parent cgroup（真正称为domain cgroup）只有在自己的层次上没有任何进程的情况下才能为其子代启用controllers。这意味着我们可以在 cgroup 目录内创建一个子树，但在写入 cgroup.subtree_control 之前，父 cgroup.procs 中列出的所有 pids 必须被迁移到子树上。这就要求所有进程必须依赖在子树下，因此不可能在非子树目录下有pids。

## 遵循 systemd 规则

systemd是目前使用最广泛的init机制。由于这个原因，Slurm需要找到一种与systemd规则共存的方法。systemd的设计者设想了一个新的规则，叫做 "single-writer"规则，这意味着每个cgroup只有一个所有者，其他人不得向其写入。详情请见 [systemd.io Cgroup Delegation 文档](https://systemd.io/CGROUP_DELEGATION/)。在实践中，这意味着在内核启动时启动的 systemd 守护进程（pid 1）将认为自己是整个 cgroup 树的绝对所有者和单一写入者。这意味着 systemd 希望其他进程不要直接修改任何 cgroup，也不要在 systemd 不知道的情况下创建目录或移动 pids。

有一种方法可以让Slurm顺利工作，那就是在systemd单元中启动Slurm守护进程，并使用特殊的systemd选项Delegate=yes。在systemd单元中启动slurmd，会给Slurm在文件系统中提供一个 "授权 "的cgroup子树，它可以在那里创建目录、移动pids，并管理自己的层次结构。实际情况是，systemd 在其内部数据库中注册了一个新的单元，并将 cgroup 目录与该单元相关联。然后，对于 cgroup 树的任何未来的 "侵入性 "操作，systemd 将有效地忽略 delegated目录。

这与cgroup v1中的情况类似，因为这不是一个内核规则，而是一个systemd规则。但这一事实与新的cgroup v2规则相结合，迫使Slurm选择一种与两者共存的设计。

### 真正的问题：systemd和重启slurmd

在为Slurm设计cgroup/v2插件时，最初的想法是让slurmd在自己的cgroup目录中设置所需的层次结构。然后它将放置作业和步骤，并将较新的分叉slurmstepds移动到相应的目录中。

这很好，直到我们需要重新启动slurmd。由于层次结构已经创建，slurmd的重启只是终止了slurmd进程，然后启动了一个新的进程，但它会尝试将新进程直接放在特定组树的根部。由于这个目录现在是一个domain controller，而不是一个子树，systemd 将无法启动守护进程。

由于 systemd 中没有任何机制来处理这种情况，我们只能将 slurmd 和分叉的 slurmstepds 分离到不同的子树目录下。由于 systemd 的设计规则是作为树上的单一写入者，所以不可能从 slurmd 或 slurmstepd 本身做一个 "mkdir"，然后将 stepd 进程移到一个新的独立目录中，这意味着这个目录不受 systemd 控制，会造成问题。

mkdir "工作的唯一方法是在一个 "委托的 "cgroup子树内完成，所以我们需要找到一个 "Delegate=yes "的单元，与slurmd的单元不同，这将保证我们的独立性。所以，我们确实需要为用户工作启动一个新的单元。

实际上，在 systemd 中，有两种类型的单元可以获得 "Delegate=yes "的参数，它们与 cgroup 目录直接相关。一种是 "Service"，另一种是 "Scope"。我们对 "Scope"感兴趣。

- Systemd Scope：systemd接收一个pid作为参数，创建一个cgroup目录，然后将提供的pid添加到该目录中。这个范围会一直保留到这个pid消失为止。


因为我们想保留任何pid的systemd作用域，所以我们需要调用systemd的dbus接口中的一个特定函数 "abandonScope"。抛弃一个作用域使得该作用域在其cgroup树中有一个活的pid时将继续存活，而不仅仅是初始pid。

值得注意的是，在与 systemd 主要开发者的讨论中，提出了 RemainAfterExit 的 systemd 参数。这个参数的目的是让单元保持活力，即使它上面的所有进程都消失了。这个选项只对 "Service "有效，对 "Scopes"无效。如果这个选项也能用于Scope，那将是一个非常有趣的选项。他们说，其功能可以扩展到不仅保留单元，而且保留cgroup目录，直到单元被手动终止。目前，单元仍然活着，但无论如何，cgroup都会被清理掉。

有了这些背景，我们准备展示用哪种解决方案来使Slurm摆脱slurmd重启的问题。

- 在slurmd启动时创建一个新的Scope，用于承载新的slurmstepd进程。它在第一次slurmd启动时做一个单一的调用。Slurmd为未来的slurmstepd pids准备了一个范围，而stepd自己在启动时也会移动到那里。这没有任何性能问题，概念上就像一个较慢的 "mkdir "+仅在第一次启动时从slurmd通知systemd。将进程从一个委托单元转移到另一个委托单元的做法得到了 systemd 开发者的认可。唯一的缺点是，scope内需要有进程，否则会终止并清理cgroup，所以slurmd需要创建一个 "sleep"的无穷大进程，我们将其编码为 "slurmstepd infinity "进程，它将永远活在作用域内。在未来，如果REMAINAfterExit参数被扩展到作用域，并允许cgroup树不被破坏，那么这个infinity进程的需求将被消除。

最后我们把 slurmd 和 slurmstepds 分开，使用一个带有 "Delegate=yes "选项的范围。

## 不遵守systemd规则的后果

有一个已知的问题是，systemd 可以决定清理 cgroup 层次结构，目的是使其与内部数据库相匹配。例如，如果系统中没有 "Delegate=yes "的单元，它就会浏览整个树状结构，并可能停用所有它认为不使用的控制器。在我们的测试中，我们停止了所有带有 "Delegate=yes "的单元，发布了 "systemd reload "或 "systemd reset-failed"，并目睹了cpuset控制器是如何从我们 "手动 "创建的cgroup树深处的目录中消失的。还有其他一些情况，而事实上systemd的开发者和文档都声称他们是树上唯一的单一写入者，这使得SchedMD决定从安全的角度出发，让Slurm与systemd共存。

值得注意的是，我们添加了IgnoreSystemd和IgnoreSystemdOnFailure作为cgroup.conf参数，这将避免与systemd的任何联系，而只是使用普通的 "mkdir "来创建相同的目录结构。这些参数仅用于开发和测试目的。

### 没有systemd的Linux发行版会怎样？

Slurm 不支持，但仍然可以工作。唯一的要求是在系统中安装libdbus、ebpf和systemd软件包来编译slurm。然后你可以在cgroup.conf中设置IgnoreSystemd参数，手动创建/sys/fs/cgroup/system.slice/目录。满足这些要求后，Slurm应该可以正常工作。

## cgroup/v2概述

我们将简单介绍一下这个插件的工作流程。

### slurmd启动

新系统：slurmd被启动。一些使用cgroup的插件（proctrack、jobacctgather或task），调用cgroup/v2插件的init()函数。这时，slurmd会使用libdbus调用dbus，并创建一个新的systemd "范围"。这个范围的名字是预定义的，根据SYSTEM_CGSLICE下的内部常量SYSTEM_CGSCOPE来设置。基本上，它最终的名字是 "slurmstepd.scope "或 "nodename_slurmstepd.scope"，这取决于Slurm在编译时是否使用了--enable-multiple-slurmd（前缀为节点名称）。与这个范围相关的cgroup目录将被固定为。"/sys/fs/cgroup/system.slice/slurmstepd.scope" 或 "/sys/fs/cgroup/system.slice/nodename_slurmstepd.scope"。

该作用域也被 "放弃"，调用dbus的 "abandonScope "方法，其目的在本页前面解释过。

由于调用dbus的 "startTransientUnit "需要一个pid作为参数，slurmd需要fork一个 "slurmstepd infinity "并使用这个参数作为参数。

对dbus的调用是异步的，所以slurmd将消息传递到Dbus总线上，然后开始主动等待，等待范围目录出现。如果目录在一个硬编码的超时内没有出现，它就会失败。否则它继续，slurmd 然后为新的 slurmstepds 和最近创建的范围目录中的 infinity pid 创建一个目录，称为 "系统"。它把 infinity 进程移到那里，然后在新的 cgroup 目录中启用所有需要的控制器。

由于这是一个常规的 systemd 单元，这个范围会在 "systemctl list-unit-files "和其他 systemd 命令中显示出来。

```
]$ systemctl cat gamba1_slurmstepd.scope
# /run/systemd/transient/gamba1_slurmstepd.scope
# This is a transient unit file, created programmatically via the systemd API. Do not edit.
[Scope]
Delegate=yes
TasksMax=infinity
]$ systemctl list-unit-files gamba1_slurmstepd.scope
UNIT FILE               STATE     VENDOR PRESET
gamba1_slurmstepd.scope transient -

1 unit files listed.

]$ systemctl status gamba1_slurmstepd.scope
● gamba1_slurmstepd.scope
     Loaded: loaded (/run/systemd/transient/gamba1_slurmstepd.scope; transient)
  Transient: yes
     Active: active (abandoned) since Wed 2022-04-06 14:17:46 CEST; 2h 47min ago
      Tasks: 1
     Memory: 1.6M
        CPU: 258ms
     CGroup: /system.slice/gamba1_slurmstepd.scope
             └─system
               └─113094 /home/lipi/slurm/master/inst/sbin/slurmstepd infinity

apr 06 14:17:46 llit systemd[1]: Started gamba1_slurmstepd.scope.
```

slurmd init的另一个动作是检测系统中哪些控制器是可用的（在/sys/fs/cgroup中），并递归地启用需要的控制器，直到达到其级别。它将为最近创建的slurmstepd范围启用它们。

```
]$ cat /sys/fs/cgroup/system.slice/gamba1_slurmstepd.scope/cgroup.controllers
cpuset cpu io memory pids

]$ cat /sys/fs/cgroup/system.slice/gamba1_slurmstepd.scope/cgroup.subtree_control
cpuset cpu memory
```

如果资源专业化被启用，slurmd也会在自己的层面上设置其内存和/或cpu约束。

### slurmd重启

Slurmd像往常一样重新启动。当重新启动时，它将检测 "scope "目录是否已经存在，如果存在，它将不做任何事情。否则它将尝试重新设置范围。

### slurmstepd启动

当需要创建一个新的步骤时，不管是作为新工作的一部分还是作为现有工作的一部分，slurmd将在它自己的cgroup目录中分叉slurmstepd进程。slurmstepd将立即开始初始化，并且（如果cgroup插件被启用）它将推断范围目录并将自己移到 "等待 "区域，也就是/sys/fs/cgroup/system.slice/slurmstepd_nodename.scope/system目录。它将立即初始化作业和步骤cgroup目录，并将自己移入其中，根据需要设置subtree_controllers。

### 终止和清理

当一个作业结束时，slurmstepd将负责删除所有创建的目录。slurmstepd.scope目录不会被Slurm删除或停止，"slurmstepd infinity "进程也不会被Slurm杀死。

当 slurmd 结束时（因为在支持的系统上，它已经被 systemd 启动了），它的 cgroup 将只是被 systemd 清理。

### 层次结构概述

层次结构将采取这种形式：

![image-20220914182841846](https://yhblog-1254039996.cos.ap-guangzhou.myqcloud.com/img-blog/image-20220914182841846.png)

图1. Slurm cgroup v2 层次结构。
左边是slurmd服务，它和systemd一起启动，单独存在于它自己的委托的cgroup中。

右边是slurmstepd的范围，它是cgroup树中的一个目录，也是所有slurmstepd和用户工作的所在。slurmstepd最初被迁移到等待新stepds的区域，系统目录，并且立即，当它初始化作业层次时，它将把自己移到相应的job_x/step_y/slurm_processes目录。

用户进程将由slurmstepd生成，并移到相应的任务目录中。

在这一点上，应该可以通过发出这个命令来检查哪些进程正在slurmstepd的范围内运行。

```
]$ systemctl status slurmstepd.scope
● slurmstepd.scope
     Loaded: loaded (/run/systemd/transient/slurmstepd.scope; transient)
  Transient: yes
     Active: active (abandoned) since Wed 2022-04-06 14:17:46 CEST; 2min 47s ago
      Tasks: 24
     Memory: 18.7M
        CPU: 141ms
     CGroup: /system.slice/slurmstepd.scope
             ├─job_3385
             │ ├─step_0
             │ │ ├─slurm
             │ │ │ └─113630 slurmstepd: [3385.0]
             │ │ └─user
             │ │   └─task_0
             │ │     └─113635 /usr/bin/sleep 123
             │ ├─step_extern
             │ │ ├─slurm
             │ │ │ └─113565 slurmstepd: [3385.extern]
             │ │ └─user
             │ │   └─task_0
             │ │     └─113569 sleep 100000000
             │ └─step_interactive
             │   ├─slurm
             │   │ └─113584 slurmstepd: [3385.interactive]
             │   └─user
             │     └─task_0
             │       ├─113590 /bin/bash
             │       ├─113620 srun sleep 123
             │       └─113623 srun sleep 123
             └─system
               └─113094 /home/lipi/slurm/master/inst/sbin/slurmstepd infinity
```

注意：如果在带有--enable-multiple-slurmd的开发系统上运行，slurmstepd.scope将有节点名称的预置。

## 在任务层面上工作

在用户工作层次中，有一个名为task_special的目录。jobacctgather/cgroup 和 task/cgroup 插件分别在任务层获取统计数据和约束资源。其他插件如proctrack/cgroup只是在步骤层工作。为了统一层次结构并使其适用于所有不同的插件，当一个插件要求将一个pid添加到一个步骤而不是一个任务时，这个pid将被放入一个特殊的目录，称为task_special。如果另一个插件将这个pid添加到一个任务中，它将从那里被迁移。通常情况下，当调用proctrack_g_add_pid向一个步骤添加pid时，proctrack插件会发生这种情况。

## 基于eBPF的设备控制器

在cgroup v2中，设备控制器接口已被删除。现在需要创建一个BPF_PROG_TYPE_CGROUP_DEVICE类型的bpf程序，并将其附加到所需的cgroup，而不是通过文件来控制它。这个程序由slurmtepd动态创建，并通过bpf syscall插入内核，它描述了作业、步骤和任务中允许或拒绝的设备。

唯一被管理的设备是gres.conf文件中描述的设备。

这种程序的插入和移除将被记录在系统日志中。

```
apr 06 17:20:14 node1 audit: BPF prog-id=564 op=LOAD
apr 06 17:20:14 node1 audit: BPF prog-id=565 op=LOAD
apr 06 17:20:14 node1 audit: BPF prog-id=566 op=LOAD
apr 06 17:20:14 node1 audit: BPF prog-id=567 op=LOAD
apr 06 17:20:14 node1 audit: BPF prog-id=564 op=UNLOAD
apr 06 17:20:14 node1 audit: BPF prog-id=567 op=UNLOAD
apr 06 17:20:14 node1 audit: BPF prog-id=566 op=UNLOAD
apr 06 17:20:14 node1 audit: BPF prog-id=565 op=UNLOAD
```

## 用不同的cgroup版本运行不同的节点

要使用的cgroup版本完全取决于节点。正因为如此，有可能在不同的节点上用不同的cgroup插件运行同一个作业。配置是在每个节点的cgroup.conf中完成的。

不能做的是在不重启和配置节点的情况下交换cgroup.conf中cgroup插件的版本。因为我们不支持混合控制器版本的 "混合 "系统，一个节点必须以一个特定的cgroup版本启动。

## 配置

在配置方面，设置与之前的cgroup/v1插件没有太大区别，但在cgroup.conf中配置cgroup插件时，必须考虑到以下因素。

### Cgroup 插件

这个选项允许系统管理员指定在节点上运行哪个cgroup版本。建议使用autodetect并忘记它，但也可以强制使用插件版本。

**CgroupPlugin=[autodetect|cgroup/v1|cgroup/v2] 。**

### 开发者选项

- IgnoreSystemd=[yes|no]。该选项用于避免调用dbus来联系systemd。slurmd启动时不会请求创建一个新的作用域，而只会使用 "mkdir "为slurmstepds准备cgroup目录。由于上述原因，不支持在装有 systemd 的生产系统中使用该选项。不过这个选项对没有systemd的系统还是很有用的。
- IgnoreSystemdOnFailure=[yes|no]。该选项将在不创建systemd "范围 "的情况下，退回到手动模式创建cgroup目录。只有在调用dbus时返回错误时才会这样，就像使用IgnoreSystemd一样。
- CgroupAutomount=[yes|no]。该选项仅在设置了IgnoreSystemd时使用。如果两者都设置了，slurmd 将检查 /sys/fs/cgroup 中所有可用的控制器，并递归地启用它们，直到达到 slurmd 的水平。这将意味着手动创建的slurmstepd目录也将设置这些控制器。
- CgroupMountPoint=/path/to/mount/point：在大多数使用cgroup v2的情况下，这个参数不应该被使用，因为/sys/fs/cgroup将是唯一的cgroup目录。

### 忽略的参数

由于 Cgroup v2 在内存控制器中不再提供 Kmem* 或 swappiness 接口，cgroup.conf 中的下列参数将被忽略。

```
AllowedKmemSpace=
MemorySwappiness=
MaxKmemPercent=
MinKmemSpace=
```

## 要求

对于构建cgroup/v2，有两个必要的库在配置时被检查。在配置时查看你的config.log，看看它们是否在你的系统上被正确检测到。

| Library | Header file          | Package provides | Configure option | Purpose                              |
| ------- | -------------------- | ---------------- | ---------------- | ------------------------------------ |
| eBPF    | include/linux/ebpf.h | kernel-headers   | --with-ebpf=     | Constrain devices to a job/step/task |
| dBus    | dbus-1.0/dbus/dbus.h | dbus-devel       | n/a              | dBus API for contacting systemd      |

**注意：**在没有systemd的系统中，编译Slurm也需要这些库。如果存在其他要求，比如不包括dbus或systemd包的要求，则必须修改配置文件。

## cgroup v2上的PAM Slurm Adopt插件

pam_slurm_adopt插件与cgroup/v1的API有依赖关系，因为在某些情况下，它依赖于作业的cgroup创建时间来选择哪个作业id应该被加入你的sshd pid。在v2版本中，我们希望消除这种依赖性，不依赖cgroup文件系统，而只是依赖作业ID。这并不能保证 sshd 会话被插入最年轻的作业中，但可以保证它被放入最大的作业 ID 中。由于这个原因，我们消除了插件对特定cgroup层次的依赖。