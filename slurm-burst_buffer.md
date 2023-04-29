---
title: slurm--Slurm Burst Buffer 指南
description: slurm中文翻译系列，机翻后纠正了一点，发现其他错误望指出，来源：https://github.com/SchedMD/slurm/blob/master/doc/html/burst_buffer.shtml
published: false
date: 2023-04-29T09:42:18.700Z
tags: slurm
editor: markdown
dateCreated: 2023-04-16T14:04:52.625Z
---

## 概述

本指南解释了如何使用Slurm突发缓冲器插件。在适当的地方，它解释了这些插件的工作原理，以指导如何最好地使用这些插件。

Slurm突发缓冲区插件在作业生命周期的不同点上调用一个脚本。

1. - 在作业提交时
1. - 当作业的估计开始时间确定后，作业处于等待状态。这被称为 `"stage-in"`。
1. - 一旦作业被安排好，但还没有开始运行。这被称为 `"pre-run"`。
1. - 一旦作业已经完成或被取消，但Slurm还没有为作业释放资源。这被称为 `"stage-out"`。
1. - 一旦作业已经完成，并且Slurm已经释放了作业的资源。这就是所谓的 `"teardown"`。

这个脚本在slurmctld节点上运行。这些是支持的插件：

- datawarp
- lua

### Datawarp

这个插件提供了与Cray的Datawarp APIs挂钩的功能。Datawarp实现了突发缓冲区，它是一种共享的高速存储资源。Slurm提供了对分配这些资源的支持，将文件存入，为使用这些资源的作业调度计算节点，以及将文件取出。突发缓冲区也可以在作业的有效期内用作临时存储，而不需要文件分流。另一个典型的用例是持久性存储，不与任何特定作业相关联。

### Lua

这个插件提供钩子到一个由Lua脚本定义的API。开发这个插件是为了向系统管理员提供一种方法，在作业生命周期的不同点上完成任何任务（不仅仅是文件暂存）。这些任务可能包括文件暂存、节点维护或任何其他希望在上述五种作业状态中的一种或多种运行的任务。

突发缓冲区API将只为特别要求使用它们的作业而被调用。作业提交命令部分解释了作业如何请求使用突发缓冲区API。

## 配置（针对系统管理员）

### 普通配置

- 要启用突发缓冲区插件，在`slurm.conf`中设置`BurstBufferType`。如果没有设置，那么将不会加载突发缓冲区插件。只能指定一个突发缓冲区插件。
- 在`slurm.conf`中，你可以设置`DebugFlags=BurstBuffer`，以便从突发缓冲区插件中获得详细的日志记录。这将导致非常粗略的日志记录，不打算在生产系统中长期使用，但这可能对调试有用。
- 突发缓冲区的TRES限制可以通过关联或QOS来配置，就像为节点、CPU或任何GRES配置TRES限制一样。要使Slurm跟踪突发缓冲区资源，在`slurm.conf`中的`AccountingStorageTres`中添加`bb/datawarp`（用于datawarp插件）或`bb/lua`（用于lua插件）。
- 作业的突发缓冲区需求的大小可以作为设置作业优先级的一个因素，如多因素优先级文件中所述。突发缓冲区资源部分解释了这些资源是如何定义的。
- Burst-buffer的特定配置可以在`burst_buffer.conf`中设置。配置设置包括诸如哪些用户可以使用突发缓冲区、超时、突发缓冲区脚本的路径等内容。更多信息请参见`burst_buffer.conf`手册。
- 必须安装JSON-C库才能构建Slurm的`burst_buffer/datawarp`和`burst_buffer/lua`插件，它们必须解析JSON格式数据。详情请见Slurm的JSON安装信息。

### Datawarp

`slurm.conf`:

```
BurstBufferType=burst_buffer/datawarp
```

datawarp插件调用两个脚本：

- **dw_wlm_cli** - Slurm `burst_buffer/datawarp`插件调用这个脚本来执行突发缓冲区功能。它应该是由Cray提供的。这个脚本的位置是由`burst_buffer.conf`中的`GetSysState`定义的。这个脚本的模板是由Slurm提供的。

  ```
  src/plugins/burst_buffer/datawarp/dw_wlm_cli
  ```

- **dwstat** - Slurm `burst_buffer/datawarp`插件调用这个脚本以获得状态信息。它应该是由Cray提供的。这个脚本的位置由`burst_buffer.conf`中的`GetSysStatus`定义。Slurm中提供了这个脚本的模板。

  ```
  src/plugins/burst_buffer/datawarp/dwstat
  ```



### Lua

slurm.conf:

```
BurstBufferType=burst_buffer/lua
```

lua插件调用一个必须命名为`burst_buffer.lua`的单一脚本。这个脚本需要与`slurm.conf`存在于同一个目录中。以下函数必须存在，尽管它们可能什么都不做，只是返回成功。

- slurm_bb_job_process
- slurm_bb_pools
- slurm_bb_job_teardown
- slurm_bb_setup
- slurm_bb_data_in
- slurm_bb_real_size
- slurm_bb_paths
- slurm_bb_pre_run
- slurm_bb_post_run
- slurm_bb_data_out
- slurm_bb_get_status

Slurm提供了一个burst_buffer.lua的模板：`etc/burst_buffer.lua.example`

这个模板记录了更多关于函数的细节，比如所需的参数，每个函数何时被调用，每个函数的返回值，以及一些简单的例子。

## Lua实现

本节的目的是提供有关Lua插件的额外信息，以帮助那些希望实现Lua API的系统管理员。本节中最重要的几点是：

- `burst_buffer.lua`中的一些函数必须快速运行，不能被杀死；其余的函数允许根据需要运行很长时间，可以被杀死。
- 最多允许512份`burst_buffer.lua`同时运行，以避免超过系统限制。

### `burst_buffer.lua`是如何运行的？

Lua脚本可以通过`fork()`和`exec()`系统调用在一个单独的进程中运行，也可以通过Lua的C API在现有进程中调用。lua插件的目标之一是避免从slurmctld内部调用`fork()`，因为它可能严重损害slurmctld的性能。datawarp插件在每次突发缓冲区API调用时都会从slurmctld调用`fork()`和`exec()`，而这已经被证明会严重损害slurmctld的性能。因此，slurmctld使用Lua的C API调用`burst_buffer.lua`，而不是使用`fork()`。

`burst_buffer.lua`中的一些函数被允许运行很长时间，但是如果作业被取消，如果slurmctld被重新启动，或者运行时间超过`burst_buffer.conf`中配置的超时，它们可能需要被杀死。然而，通过Lua的C API对Lua脚本的调用不能在同一进程中被杀死；只有杀死调用Lua脚本的整个进程才能杀死Lua脚本。

为了解决这种情况，`burst_buffer.lua`被以两种不同的方式调用。

- 从slurmctld调用`slurm_bb_job_process`、`slurm_bb_pools`和`slurm_bb_paths`函数。由于上面的解释，运行这些函数之一的脚本不能被杀死。由于这些函数是在slurmctld持有一些mutexes的情况下被调用的，如果它们的速度很慢，将对slurmctld的性能和响应性造成极大的伤害。因为直接调用这些函数比调用`fork()`来创建一个新进程要快，这被认为是可以接受的权衡。因此，**这些函数不能被杀死。**
- `burst_buffer.lua`中的其余函数能够运行更长时间而不产生不利影响。这些函数需要能够被杀死。这些函数是由一个叫做slurmscriptd的轻量级Slurm守护程序调用的。每当这些函数之一需要运行时，slurmctld告诉slurmscriptd运行该函数；然后slurmscriptd调用`fork()`来创建一个新进程，然后调用相应的函数。这就避免了从slurmctld调用`fork()`，同时仍然提供了一种在需要时杀死正在运行的`burst_buffer.lua`副本的方法。因此，这些函数可以被杀死，**如果它们运行的时间超过了`burst_buffer.conf`中配置的适当的超时值，它们将被杀死。**

每个函数的调用方式也记录在burst_buffer.lua.example文件中。

### 限制条件

每个通过slurmscriptd运行的脚本副本都在一个新的进程中运行，并且打开一个管道（也就是一个文件）来读取脚本的响应。为了避免超过打开进程或打开文件的限制、内存耗尽或超过其他系统的限制，每个 `"stage"`最多允许128个脚本副本同时运行，其中的阶段包括进入阶段、运行前阶段、退出阶段和关闭阶段（关于突发缓冲区阶段的更多信息请参见突发缓冲区状态部分）。这意味着，一次最多可以运行512份`burst_buffer.lua`。如果没有这个限制，作业的吞吐量就会降低，而且测试证实，可能会超过进程开放文件的限制。

**警告：**不要在`burst_buffer.lua`中安装信号处理程序，因为它是由slurmctld直接调用的。如果slurmctld收到一个信号，它可能会试图从`burst_buffer.lua`中运行信号处理程序，甚至在调用`burst_buffer.lua`完成之后，这将导致崩溃。

## 突发缓冲区资源

突发缓冲区API可以定义突发缓冲区资源 `"pools"`，作业可以从中请求一定数量的池空间。如果一个池子没有足够的空间来满足一个作业的请求，该作业将保持挂起，直到池子有足够的空间。一旦池子有足够的空间，Slurm就可以开始为该作业进行分期。当分阶段开始时，Slurm从池的可用空间中减去作业的请求空间。当拆解完成后，Slurm 将作业要求的空间加回到池的可用空间中。作业提交命令部分解释了作业如何从池中请求空间。池的空间是一个标量。

### Datawarp

- pool是由`dw_wlm_cli`定义的，代表字节数。这个脚本打印一个JSON格式的字符串，定义池子到stdout。
- 如果一个作业没有请求一个pool，那么将使用`burst_buffer.conf`中`DefaultPool`定义的池。如果一个作业没有请求一个pool，并且`DefaultPool`没有被定义，那么这个作业将被拒绝。

### Lua

- pools在这个插件中是可选的，可以代表任何东西。
- `burst_buffer.conf`中的`DefaultPool`在这个插件中不使用。
- pools是由`burst_buffer.lua`在函数slurm_bb_pools中定义的。如果不需要池子，那么这个函数应该只返回slurm.SUCCESS。如果需要pool，那么这个函数应该返回两个值：（1）slurm.SUCCESS，和（2）一个定义pool的JSON格式的字符串。`burst_buffer.lua.example`中提供了一个例子。目前JSON字符串中的有效字段是：
  - id - 一个定义池子名称的字符串
  - quantity - 一个数字，定义了池中的空间数量
  - granularity - 一个数字，定义了可以从这个池中分配的最低分辨率的空间。如果作业请求的数字不是颗粒度的倍数，那么作业的请求将被四舍五入为颗粒度的最近倍数。例如，如果granularity等于1000，那么可以从这个池子里为一个作业分配的最小空间是1000。如果一个作业从这个池子里请求的单位少于1000，那么这个作业的请求将被四舍五入为1000。



## 作业提交命令

正常的操作模式是批处理作业在批处理脚本中指定突发缓冲区要求。含有特定指令的批处理脚本行（取决于正在使用的插件）将通知Slurm，它应该为该作业运行突发缓冲区阶段。这些行也将描述该作业的突发缓冲区要求。

`salloc`和`srun`命令可以用`-bb`和`-bbf`选项指定突发缓冲区要求。这在命令行作业选项部分有描述。

所有的突发缓冲区指令都应该在批处理脚本的顶部的注释中指定。它们可以放在任何`#SBATCH`指令之前、之后或穿插在其中。所有突发缓冲区阶段都发生在作业生命周期的特定点上，如概述部分所述；它们不会在作业执行期间发生。例如，所有的持久性突发缓冲区（仅由datawarp插件使用）的创建和删除发生在作业的计算部分发生之前。类似地，你不能在脚本执行的不同点上运行stage-in；突发缓冲区的stage-in在作业开始前进行，stage-out则在作业完成后进行。

对于这两个插件，一个作业可以从一个突发缓冲区资源池中请求一定的空间（大小或容量）。

- **pool**的规格只是一个字符串，与池的名称相匹配。例如：`pool=pool1`
- **容量**规格是一个数字，表示从池中需要的空间量。容量规格可以包括后缀 "N"（节点）、"K|KiB"、"M|MiB"、"G|GiB"、"T|TiB"、"P|PiB"（1024的幂）和 "KB"、"MB"、"GB"、"TB "和 "PB"（1000的幂）。注意：通常Slurm把KB、MB、GB、TB、PB、TB单位解释为1024的幂，但是对于Burst Buffers大小规格，Slurm支持IEC/SI两种格式。这是因为CRAY的API支持这两种格式。

在作业提交时，Slurm会执行基本的指令验证，也会在突发缓冲区脚本中运行一个函数。这个函数可以对作业脚本中使用的指令进行验证。如果Slurm确定选项无效，或者如果突发缓冲脚本返回错误，作业将被拒绝，并直接向用户返回错误信息。

请注意，为了支持向后兼容，未被识别的选项可能被忽略（即在某些版本的Slurm识别的选项，但不被其他版本识别的情况下，作业提交不会失败）。如果作业被接受，但后来失败了（例如，一些问题的暂存文件），作业将被保留，其 "原因 "字段将被设置为由底层基础设施提供的错误信息。

用户也可以使用--mail-type=stage_out或--mail-type=all选项，要求在突发缓冲区阶段性淘汰完成后得到电子邮件通知。电子邮件的主题行将是这样的形式。

```
SLURM Job_id=12 Name=my_app Staged Out, StageOut time 00:05:07
```

下面的插件小节给出了每个插件的额外信息，并提供了作业脚本的例子。命令行示例在命令行作业选项部分给出。

### Datawarp

当使用`burst_buffer/datawarp`插件时，`#DW`（代表 "DataWarp"）的指令用于突发缓冲器指令。关于DataWarp选项的细节，请参考Cray文档。对于DataWarp系统，#BB的指令可以用来创建或删除持久的突发缓冲区存储。
**注意：**使用`#BB`指令是因为该指令是由Slurm解释的，而不是由Cray Datawarp软件解释的。这将在持久性突发缓冲区一节中详细讨论。

对于特定作业的突发缓冲区，需要指定突发缓冲区的容量。如果作业没有指定容量，那么该作业将被拒绝。一个作业也可以指定它想要的资源池；如果作业没有指定一个池，那么将使用burst_buffer.conf中DefaultPool指定的池（如果配置了）。

下面的作业脚本从默认池中请求突发缓冲区资源，并请求文件被分阶段输入和分阶段输出。

```shell
#!/bin/bash
#DW jobdw type=scratch capacity=1GB access_mode=striped,private pfs=/scratch
#DW stage_in type=file source=/tmp/a destination=/ss/file1
#DW stage_out type=file destination=/tmp/b source=/ss/file1
srun application.sh
```

### Lua

这个插件的默认指令是#BB_LUA。这个插件使用的指令可以通过设置burst_buffer.conf中的Directive选项来改变。由于指令必须总是以#号开始（在shell脚本中开始一个注释），这个选项应该只指定#号后面的字符串。例如，如果burst_buffer.conf包含以下内容。

```
Directive=BB_EXAMPLE
```

那么突发缓冲区的指令将是`#BB_EXAMPLE`。

如果在burst_buffer.conf中没有指定`Directive`选项，那么将使用这个插件的默认指令（`#BB_LUA`）。

由于这个插件被设计成通用和灵活的，这个插件只需要给出指令。如果给出了指令，Slurm将运行作业的所有突发缓冲阶段。

为作业运行所有突发缓冲阶段所需的最小信息的例子。

```
#!/bin/bash
#BB_LUA
srun application.sh
```

因为突发缓冲区池对这个插件来说是可选的（见突发缓冲区资源部分），作业不需要指定池或容量。如果池是由突发缓冲区API提供的，那么作业可以请求一个池和容量。

```
#!/bin/bash
#BB_LUA pool=pool1 capacity=1K
srun application.sh
```

一个作业可以选择是否指定一个池。如果一个作业没有指定一个池子，那么这个作业仍然被允许运行，并且突发缓冲阶段仍然会为这个作业运行（只要突发缓冲指令被给出）。如果作业指定了一个池，但没有找到该池，那么该作业将被拒绝。

系统管理员可以在burst_buffer.lua的slurm_bb_job_process函数中验证突发缓冲区选项。这可能包括要求作业指定一个池或验证系统管理员决定实施的任何额外选项。

## 持久突发缓冲区的创建和删除指令

本节仅适用于datawarp插件，因为持久性突发缓冲区不用于任何其他突发缓冲区插件。

这些选项用于创建和删除持久性突发缓冲区：

- `#BB create_persistent name=<name> capacity=<number> [access=<access>] [pool=<pool> [type=<type>]`
- `#BB destroy_persistent name=<name> [hurry]`

用于创建和删除持久性突发缓冲区的选项。

- name - 持久性突发缓冲区的名称不能以数字开头（数字名称保留给特定作业突发缓冲区）。
- capacity - 在作业提交命令部分有描述。
- pool - 在作业提交命令部分有描述。
- access - 访问参数确定了缓冲区的访问模式。datawarp插件支持的访问模式包括。
  - striped
  - private
  - ldbalance
- type - 类型参数标识了缓冲区的类型。datawarp插件支持的类型模式包括。
  - cache
  - scratch

在一个作业中可以创建或删除多个持久性突发缓冲区。

例子 - 创建两个持久性突发缓冲区：

```shell
#!/bin/bash
#BB create_persistent name=alpha capacity=32GB access=striped type=scratch
#BB create_persistent name=beta capacity=16GB access=striped type=scratch
srun application.sh
```

例子 - 删除两个持久性突发缓冲区

```shell
#!/bin/bash
#BB destroy_persistent name=alpha
#BB destroy_persistent name=beta
srun application.sh
Persistent burst buffers 
```

持久的突发缓冲区可以由一个不需要计算资源的作业来创建和删除。提交一个具有所需突发缓冲区指令的作业，并指定节点数为零（例如，`sbatch -N0 setup_buffers.bash`）。试图提交一个没有突发缓冲区指令的零尺寸作业，或者提交作业特定的突发缓冲区指令，将产生一个错误。请注意，零尺寸作业不支持作业阵列或异质作业分配。

注意：创建和销毁持久性突发缓冲区的能力可能受到`burst_buffer.conf`文件中`Flags`选项的限制。更多信息请参见`burst_buffer.conf` man页。默认情况下，只有特权用户（即Slurm操作员和管理员）可以创建或销毁持久性突发缓冲区。

## 命令行作业选项

除了在批处理脚本中放置突发缓冲区指令外，命令行选项`-bb`和`-bbf`也可以包括突发缓冲区指令。这些命令行选项可用于`salloc`、`sbatch`和`srun`。注意，`--bb`选项不能创建或销毁持久的突发缓冲区。

`--bbf`选项需要一个文件名作为参数，该文件应该包含一个与批处理作业相同的突发缓冲区操作集合。

另外，可以使用`--bb`选项来指定突发缓冲区指令作为选项参数。这个选项的行为取决于使用哪种突发缓冲区插件。当使用`--bb`选项时，Slurm解析这个选项并创建一个临时的突发缓冲器脚本文件，由突发缓冲器插件内部使用。

### Datawarp

当使用 `--bb` 选项时，指令的格式既可以与批处理脚本中使用的相同，也可以使用一组非常有限的选项，这些选项会被翻译成等价的脚本，以便以后处理。下列选项是允许的：

- access=<access>
- capacity=<number>
- swap=<number>
- type=<type>
- pool=<name>

多个选项应该用空格分隔。如果指定了交换选项，作业还必须指定所需的节点数。

例子：

```
# Sample execute line:
srun --bb="capacity=1G access=striped type=scratch" a.out

# Equivalent script as generated by Slurm's burst_buffer/datawarp plugin
#DW jobdw capacity=1GiB access_mode=striped type=scratch
```

### Lua

这个插件不对`--bb`选项给出的突发缓冲区指令做任何特殊的解析或翻译。当使用`--bb`选项时，其格式与批处理脚本相同： Slurm只强制要求必须指定burst buffer指令。参见作业提交命令的Lua小节中的其他信息。

例子：

```
# Sample execute line:
srun --bb="#BB_LUA pool=pool1 capacity=1K"

# Equivalent script as generated by Slurm's burst_buffer/datawarp plugin
#BB_LUA pool=pool1 capacity=1K
```

## 符号替换

Slurm支持一些符号，可用于自动填写某些作业细节，例如，使阶段性或阶段性目录路径随每个作业提交而变化。

| %%   | %                                   |
| ---- | ----------------------------------- |
| `%A` | Array Master Job Id                 |
| `%a` | Array Task Id                       |
| `%d` | Workdir                             |
| `%j` | Job id                              |
| `%u` | User Name                           |
| `%x` | Job Name                            |
| `\\` | Stop further processing of the line |

## 状态命令

Slurm跟踪的突发缓冲区信息可以通过使用`scontrol show burst`命令或使用`sview`命令的Burst Buffer标签来获得。示例如下。

Datawarp插件例子：

```
$ scontrol show burst
Name=datawarp DefaultPool=wlm_pool Granularity=200GiB TotalSpace=5800GiB FreeSpace=4600GiB UsedSpace=1600GiB
  Flags=EmulateCray
  StageInTimeout=86400 StageOutTimeout=86400 ValidateTimeout=5 OtherTimeout=300
  GetSysState=/home/marshall/slurm/master/install/c1/sbin/dw_wlm_cli
  GetSysStatus=/home/marshall/slurm/master/install/c1/sbin/dwstat
  Allocated Buffers:
    JobID=169509 CreateTime=2021-08-11T10:19:06 Pool=wlm_pool Size=1200GiB State=allocated UserID=marshall(1017)
    JobID=169508 CreateTime=2021-08-11T10:18:46 Pool=wlm_pool Size=400GiB State=staged-in UserID=marshall(1017)
  Per User Buffer Use:
    UserID=marshall(1017) Used=1600GiB
```

lua插件例子：

```
$ scontrol show burst
Name=lua DefaultPool=(null) Granularity=1 TotalSpace=0 FreeSpace=0 UsedSpace=0
  PoolName[0]=pool1 Granularity=1KiB TotalSpace=10000KiB FreeSpace=9750KiB UsedSpace=250KiB
  PoolName[1]=pool2 Granularity=2 TotalSpace=10 FreeSpace=10 UsedSpace=0
  PoolName[2]=pool3 Granularity=1 TotalSpace=4 FreeSpace=4 UsedSpace=0
  PoolName[3]=pool4 Granularity=1 TotalSpace=5GB FreeSpace=4GB UsedSpace=1GB
  Flags=DisablePersistent
  StageInTimeout=86400 StageOutTimeout=86400 ValidateTimeout=5 OtherTimeout=300
  GetSysState=(null)
  GetSysStatus=(null)
  Allocated Buffers:
    JobID=169504 CreateTime=2021-08-11T10:13:38 Pool=pool1 Size=250KiB State=allocated UserID=marshall(1017)
    JobID=169502 CreateTime=2021-08-11T10:12:06 Pool=pool4 Size=1GB State=allocated UserID=marshall(1017)
  Per User Buffer Use:
    UserID=marshall(1017) Used=1000256KB
```
使用`scontrol show bbstat ...` 或`scontrol show dwstat ... `命令可以从`scontrol`访问突发缓冲器状态API。`scontrol`执行行中`bbstat`或`dwstat`后面的选项会直接传递给`bbstat`或`dwstat`命令，如下图所示。在datawarp插件中，该命令调用Cray的`dwstat`脚本。有关`dwstat`选项和输出的细节，请参见Cray Datawarp文档。在lua插件中，该命令调用`burst_buffer.lua`中的`slurm_bb_get_status`函数。示例如下。

Datawarp插件示例：
```
/opt/cray/dws/default/bin/dwstat
$ scontrol show dwstat
    pool units quantity    free gran'
wlm_pool bytes  7.28TiB 7.28TiB 1GiB'

$ scontrol show dwstat sessions
 sess state      token creator owner             created expiration nodes
  832 CA---  783000000  tester 12345 2015-09-08T16:20:36      never    20
  833 CA---  784100000  tester 12345 2015-09-08T16:21:36      never     1
  903 D---- 1875700000  tester 12345 2015-09-08T17:26:05      never     0

$ scontrol show dwstat configurations
 conf state inst    type access_type activs
  715 CA---  753 scratch      stripe      1
  716 CA---  754 scratch      stripe      1
  759 D--T-  807 scratch      stripe      0
  760 CA---  808 scratch      stripe      1
```
  
Lua插件的例子：

这个例子没有做任何有用的事情，它只是展示了这个调用是如何使用的。

```
-- Implement this function in burst_buffer.lua
function slurm_bb_get_status(...)
    local i, v, args, outstr, arr

    arr = { }
    -- Create a table from variable arg list
    args = {...}
    args.n = select("#", ...)

    for i,v in ipairs(args) do
        arr[#arr+1] = tostring(v)
    end
    outstr = table.concat(arr, "\n")

    return slurm.SUCCESS, "Status return message.\nArgs:\n" .. outstr .. "\n"
end
$ scontrol show bbstat arg1 arg2
Status return message.
Args:
arg1
arg2
```
## 预先预订
  