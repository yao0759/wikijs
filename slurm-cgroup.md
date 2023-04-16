---
title: slurm--cgroup
description: slurm中文翻译系列，机翻后纠正了一点，发现其他错误望指出，来源：https://github.com/SchedMD/slurm/blob/master/doc/html/cgroups.shtml
published: true
date: 2023-04-16T12:18:58.298Z
tags: slurm, cgroup
editor: markdown
dateCreated: 2023-03-19T15:24:42.980Z
---

# slurm--cgroup
## Control Group概述
Control Group是由内核提供的一种机制，用于分层组织进程，并以可控和可配置的方式沿层级分配系统资源。Slurm可以利用cgroup来限制不同的资源给作业、步骤和任务，并获得关于这些资源的核算。

一个cgroup为不同的资源提供不同的控制器（以前是 "子系统"）。Slurm插件可以使用这些控制器中的几个，例如：内存、cpu、设备、冷藏室、cpuset、cpuacct。每一个启用的控制器都能将资源限制在一组进程中。如果系统上没有一个控制器，那么Slurm就不能通过cgroup来约束相关的资源。

"cgroup"代表 "控制组"，从不大写。单数形式用于指定整个功能，也可作为限定词，如 "cgroup controllers"。当明确提到多个单独的控制组时，使用复数形式 "cgroups"。

Slurm支持两种cgroup模式，传统模式（cgroup v1）和统一模式（cgroup v2）。不支持混合模式，即在一个系统中同时混合使用版本1和版本2的控制器。

请参阅kernel.org文档，了解对cgroup更全面的描述。

- [内核的Cgroup v1文档](https://www.kernel.org/doc/Documentation/cgroup-v1/cgroups.txt)
- [内核的Cgroup v2文档](https://www.kernel.org/doc/html/latest/admin-guide/cgroup-v2.html)

## Slurm cgroup插件设计

关于Slurm内部Cgroup插件的扩展信息，请阅读。

- cgroup/v1 插件文档 (尚未提供)
- cgroup/v2插件文档

### 在Slurm中使用cgroup

Slurm提供了一些插件的cgroup版本。

- `proctrack/cgroup` (用于进程跟踪和管理)
- `task/cgroup` (用于在步骤和任务级别上限制资源)
- `jobacct_gather/cgroup` (用于收集统计数据)

cgroup也可用于资源专业化（将守护程序限制在核心或内存上）。

### Slurm cgroup配置概述
有几组配置选项用于Slurm cgroup。

- `slurm.conf`提供了启用cgroup插件的选项。每个插件都可以独立于其他插件而被启用或禁用。
- `cgroup.conf` 提供了所有 cgroup 插件通用的一般选项，以及仅适用于特定插件的附加选项。
- 系统级的资源专业化是通过节点配置参数启用的。

### 目前可用的 Cgroup 插件
#### proctrack/cgroup插件
`proctrack/cgroup`插件是其他proctrack插件的替代品，如`proctrack/linux`的进程跟踪和暂停/恢复能力。

`proctrack/cgroup`使用冷冻机控制器来跟踪一个作业的所有pid。它基本上把pids存储在cgroup树的一个特定层次中，并在接到指示时负责发出这些pids的信号。例如，如果用户决定取消一个作业，Slurm将通过调用proctrack插件并要求它向该作业发送SIGTERM来在内部执行这个命令。由于proctrack在cgroup中维护了所有与Slurm相关的pids的层次结构，它将很容易知道哪些pids需要被信号化。
Proctrack还可以响应查询，获得一个作业或一个步骤的所有pid的列表。
另外，在使用`proctrack/linux`时，cgroup将pid存储在一个文件（cgroup.procs）中，插件读取该文件以获得层次结构中某一部分的所有pid。例如，当使用`proctrack/cgroup`时，一个步骤有它自己的`cgroup.procs`文件，所以获得该步骤的pid是瞬间的。在`proctrack/linux`中，我们需要递归地读取/proc，以获得父级pid的所有后代。

要启用这个插件，请在`slurm.conf`中配置以下选项。

```
ProctrackType=proctrack/cgroup
```

在cgroup.conf中没有专门针对这个插件的选项，但一般的选项都适用。详情请参见 cgroup.conf man 页。

#### task/cgroup插件
任务/组插件允许将资源限制在一个作业、一个步骤或一个任务中。这是唯一能够确保分配的边界不被违反的插件。只有`jobacctgather/linux`提供了一个非常简单的机制来约束作业的内存，但它并不可靠（有一个时间窗口，作业可以超过它的限制），而且只适用于非常罕见的cgroup不可用系统。

task/cgroup提供了以下功能。

- 将作业和步骤限制在其分配的cpuset上。
- 将工作和步骤限制在特定的内存资源中。
- 将作业、步骤和任务限制在其分配的gres上，包括gpus。

任务/组插件使用cpuset、内存和设备子系统。

要启用这个插件，在slurm.conf中的TaskPlugin配置参数中加入task/cgroup。

```
TaskPlugin=task/cgroup
```

在`cgroup.conf`中，有许多关于这个插件的具体选项。一般的选项也适用。详情请参见 cgroup.conf man 页。

这个插件可以与其他任务插件叠加，例如与`task/affinity`叠加。这将允许它将资源限制在一个工作上，同时获得亲和力插件的优势（顺序并不重要）。

```
TaskPlugin=task/cgroup,task/affinity
```

#### jobacct_gather/cgroup插件
`jobacct_gather/cgroup`插件是`jobacct_gather/linux`插件的替代品，用于收集作业、步骤和任务的会计统计。
`jobacct_gather/cgroup` 使用 `cpuacct` 和 `memory cgroup 控制器`。

这个插件收集的cpu和内存统计信息与jobacct_gather/linux收集的cpu和内存统计信息所代表的资源不一样。cgroup插件只是读取一个`cgroup.stats`文件和类似的包含整个子树的pid的信息，而linux插件从`/proc/pid/stat`获取每个pid的信息，然后进行计算，因此变得比cgroup的效率低一些（在实践中认为不明显）。

要启用这个插件，请在`slurm.conf`中配置以下选项。

```
JobacctGatherType=jobacct_gather/cgroup
```

在cgroup.conf中没有专门针对这个插件的选项，但一般的选项都适用。详情请参见 cgroup.conf man 页。

### 使用 cgroup 进行资源专用化
资源特殊化可以用来在每个计算节点上保留一个核心子集或特定数量的内存，供Slurm计算节点守护程序slurmd独家使用。

Slurmstepd不受这种资源专业化的限制，因为它被认为是作业的一部分，其消耗完全取决于作业的类型。例如，一个MPI作业可以用PMI初始化许多行列，使slurmstepd消耗更多的内存。

系统级的资源专业化是通过特殊的节点配置参数来实现的。阅读`slurm.conf`和`core_spec.html`中的核心专业化，了解更多信息。

### Slurm cgroup插件
从22.05开始，Slurm支持`cgroup/v1`和`cgroup/v2`。这两个插件都有非常不同的组织层次的方式，并响应不同的设计约束。该设计是内核维护者的责任。

## cgroup/v1和cgroup/v2的主要区别
v1和v2之间的三个主要区别是：

- v2中的Unified模式

  在`cgroup/v1`中，每个控制器都有一个单独的层次结构，这意味着作业结构必须为每个启用的控制器进行复制和管理。例如，对于同一个作业，如果使用内存和冰柜控制器，我们需要在两个控制器的目录中创建相同的`slurm/uid/job_id/step_id/`层次结构。比如说

  ```
  /sys/fs/cgroup/memory/slurm/uid_1000/job_1/step_0/
  /sys/fs/cgroup/freezer/slurm/uid_1000/job_1/step_0/
  ```

  在`cgroup/v2`中，我们有一个统一的层次结构，控制器在同一级别被启用，并以不同的文件呈现给用户。

  ```
  /sys/fs/cgroup/system.slice/slurmstepd.scope/job_1/step_0/
  ```

- v2中自上而下的约束

  资源是自上而下分配的，只有当资源从父级分配到它时，cgroup才能进一步分配资源。启用的控制器在 `cgroup.controllers` 文件中列出，子树中启用的控制器在 `cgroup.subtree_control` 中列出。

- v2中的 "No-Internal-Process"约束

  在 `cgroup/v1` 中，层次结构是自由的，这意味着人们可以在树中创建任何目录并在其中放置 pids。在`cgroup/v2`中，有一个内核限制，阻碍了向非叶子目录添加pid。

- Systemd对`cgroup/v2`的依赖--slurmd和stepds的分离

  这不是内核的限制，而是systemd的决定，它对决定使用`Delegate=yes`的服务施加了一个重要限制。systemd，pid 1，决定成为cgroup层次结构的完全拥有者，`/sys/fs/cgroup`，试图强加一个`single-writer`的设计。这意味着与cgroup相关的一切都必须在systemd的控制之下。如果有人决定手动修改 cgroup 树，创建目录并移动 pids，那么 systemd 有可能在某个时刻决定启用或禁用整个树上的控制器，或者移动 pids。根据经验，一个

  ```
  systemd reload
  ```

  或者

  ```
  systemd reset-failed
  ```

  如果系统中没有任何 "systemd unit "在使用控制器，并且系统中没有任何 "`Delegate=Yes` "启动的"`systemd unit`"，则删除该树中任何级别和目录的控制器。这是因为 systemd 想清理 cgroup 树，并将其与内部单元数据库进行匹配。事实上，查看systemd的代码可以发现，与 "`Delegate=yes`"标志的单元相关的cgroup目录被忽略，而任何其他cgroup目录都被修改。这使得我们必须在 "`Delegate=yes`"的单元下启动slurmd和slurmstepd进程。这意味着我们需要用systemd来启动、停止和重启slurmd。如果我们这样做，由于我们之前可能已经修改了slurmd所属的树（例如添加了工作目录），systemd将无法重启slurmd，因为前面提到的Top-down约束。它将无法把新的slurmd pid放入根c组，而根c组现在是一个非叶子。这迫使我们将 slurmstepd 的 cgroup 层次结构与 slurmd 的层次结构分开，由于我们需要通知 systemd 并将 slurmstepd 放入一个新的单元，我们将对 systemd 进行 dbus 调用，为 slurmstepds 创建一个新的范围。更多信息请参见 systemd ControlGroupInterface。

  

下面的差异不应该影响其他插件与 cgroup 插件的交互，相反，它们只显示了内部的功能差异。

- `cgroup/v2` 中的控制器是通过写入 cgroup.controllers 启用的，而在 cgroup/v1 中，必须用文件系统类型"-t cgroup "和相应的选项（例如"-o freezer"）挂载一个新的挂载点。
- 在`cgroup/v2`中，冻结控制器本身就存在于cgroup.freeze接口中。在 cgroup/v1 中，它是一个特定的、独立的控制器，需要被安装。
- 设备控制器在`cgroup/v2`中不存在，而必须在内核中插入一个新的eBPF程序。
- 在`cgroup/v2`中，`memory.stat`文件已经改变，现在我们做的是`anon+swapcached+anon_thp`的总和，以符合v1中的RSS概念。
- 在`cgroup/v2`中，cpu.stat提供了milis指标，而`cgroup/v1`中的`puacct.stat`提供了`USER_HZ`指标。

**控制器接口之间的主要区别**

| cgroup/v1                   | cgroup/v2                             |
| --------------------------- | ------------------------------------- |
| memory.limit_in_bytes       | memory.max                            |
| memory.soft_limit_in_bytes  | memory.high                           |
| memory.memsw_limit_in_bytes | memory.swap.max                       |
| memory.swappiness           | none                                  |
| freezer.state               | cgroup.freeze                         |
| cpuset.cpus                 | cpuset.cpus.effective and cpuset.cpus |
| cpuset.mems                 | cpuset.mems.effective and cpuset.mems |
| cpuacct.stat                | cpu.stat                              |
| memory.kmem*                | none                                  |
| device.*                    | ebpf program                          |

## 其他一般情况

- 当使用`cgroup/v1`时，一些配置可以排除掉交换cgroup的核算。这个核算是由内存控制器提供的功能的一部分。如果这个功能从内核或启动参数中被禁用，试图启用swap约束将产生一个错误。如果需要这样做，在内核命令行中添加以下参数。

  ```
  cgroup_enable=memory swapaccount=1
  ```

  这通常可以放在/etc/default/grub的`GRUB_CMDLINE_LINUX`变量内。在更新该文件后，必须运行一个诸如`update-grub`的命令。这个功能也可以在内核配置中用参数禁用。

  ```
  config_memcg_swap=
  ```

- 在某些Linux发行版中，可以使用systemd参数JoinControllers，该参数实际上已被废弃。这个参数允许在`cgroup/v1`中把多个控制器挂载在一个层次结构中，或多或少试图模仿`cgroup/v2`在 "Unified"模式中的行为。然而，Slurm在这种配置下不能正常工作，所以请确保你的`system.conf`不使用JoinControllers，并且在使用`cgroup/v1`传统模式时，所有的cgroup控制器都在独立的目录下。	