---
title: slurm--工作阵列支持
description: slurm中文翻译系列，机翻后纠正了一点，发现其他错误望指出，来源：https://github.com/SchedMD/slurm/blob/master/doc/html/job_array.shtml
published: true
date: 2023-04-16T14:56:42.014Z
tags: slurm
editor: markdown
dateCreated: 2023-04-16T14:56:42.014Z
---

## 概述

作业阵列提供了一种机制，可以快速而方便地提交和管理类似的作业集合；有数百万个任务的作业阵列可以在几毫秒内提交（受配置的大小限制）。所有的作业都必须有相同的初始选项（如大小、时间限制等），然而在作业开始执行后，可以使用指定阵列的`JobID`或单个`ArrayJobID`的`scontro`l命令改变其中的一些选项。
```
$ scontrol update job=101 ...
$ scontrol update job=101_1 ...
```
作业数组只支持批处理作业，数组索引值用`sbatch`命令的`--array`或`-a`选项来指定。选项参数可以是具体的数组索引值、索引值的范围和可选的步长，如下面的例子中所示。注意，最小的索引值是0，最大值是Slurm的配置参数（`MaxArraySize`减去1）。作为工作阵列一部分的工作将有环境变量`SLURM_ARRAY_TASK_ID`设置为其阵列索引值。
```
# Submit a job array with index values between 0 and 31
$ sbatch --array=0-31    -N1 tmp

# Submit a job array with index values of 1, 3, 5 and 7
$ sbatch --array=1,3,5,7 -N1 tmp

# Submit a job array with index values between 1 and 7
# with a step size of 2 (i.e. 1, 3, 5 and 7)
$ sbatch --array=1-7:2   -N1 tmp
```
可以使用`"%"`分隔符来指定作业阵列中同时运行的任务的最大数量。例如，`"--array=0-15%4 "`将限制该作业阵列中同时运行的任务数为4。

## 作业ID和环境变量

作业阵列将有两个额外的环境变量设置。`SLURM_ARRAY_JOB_ID`将被设置为该阵列的第一个作业ID。`SLURM_ARRAY_TASK_ID`将被设置为作业阵列的索引值。`SLURM_ARRAY_TASK_COUNT`将被设置为工作数组中的任务数。`SLURM_ARRAY_TASK_MAX`将被设置为最高的作业数组索引值。`SLURM_ARRAY_TASK_MIN`将被设置为最低的作业数组索引值。例如，一个这样的作业提交
```
sbatch --array=1-3 -N1 tmp
```
将产生一个包含三个作业的作业阵列。如果`sbatch`命令的反应是
```
Submitted batch job 36
```
那么环境变量将被设置为如下：
```
SLURM_JOB_ID=36
SLURM_ARRAY_JOB_ID=36
SLURM_ARRAY_TASK_ID=1
SLURM_ARRAY_TASK_COUNT=3
SLURM_ARRAY_TASK_MAX=3
SLURM_ARRAY_TASK_MIN=1

SLURM_JOB_ID=37
SLURM_ARRAY_JOB_ID=36
SLURM_ARRAY_TASK_ID=2
SLURM_ARRAY_TASK_COUNT=3
SLURM_ARRAY_TASK_MAX=3
SLURM_ARRAY_TASK_MIN=1

SLURM_JOB_ID=38
SLURM_ARRAY_JOB_ID=36
SLURM_ARRAY_TASK_ID=3
SLURM_ARRAY_TASK_COUNT=3
SLURM_ARRAY_TASK_MAX=3
SLURM_ARRAY_TASK_MIN=1
```

所有的Slurm命令和API都识别`SLURM_JOB_ID`值。大多数命令也能识别`SLURM_ARRAY_JOB_ID`和`SLURM_ARRAY_TASK_ID`的值，用下划线隔开，以识别工作阵列的一个元素。使用上面的例子，`"37 "`或 `"36_2 "`将是识别作业36的第二个数组元素的同等方式。一套API已经被开发出来，可以在一个单一的函数调用中对整个工作阵列或工作阵列的选定任务进行操作。该函数响应由一个数组组成，用于识别工作ID的各种任务的各种错误代码。例如，`job_resume2()`函数可能返回一个错误代码数组，表明任务1和2已经完成；任务3到5被成功恢复，而任务6到99还没有开始。

## 文件名

有两个额外的选项可用于指定一个作业的`stdin`、`stdout`和`stderr`文件名：`%A`将被`SLURM_ARRAY_JOB_ID`（如上定义）的值取代，`%a`将被`SLURM_ARRAY_TASK_ID`（如上定义）的值取代。作业阵列的默认输出文件格式是 `"slurm-%A_%a.out"`。一个明确使用格式化的例子是：
```
sbatch -o slurm-%A_%a.out --array=1-3 -N1 tmp
```
这将产生这样的输出文件名 `"slurm-36_1.out"`、`"slurm-36_2.out "`和 `"slurm-36_3.out"`。如果使用这些文件名选项而不作为作业数组的一部分，那么`"%A "`将被当前作业ID取代，`"%a "`将被4,294,967,294（相当于`0xfffffe`或`NO_VAL`）取代。

## `scancel`命令的使用

如果一个工作数组的工作ID被指定为`scancel`命令的输入，那么该工作数组的所有元素将被取消。另外，也可以使用正则表达式指定一个数组ID，用于取消作业。
```
# Cancel array ID 1 to 3 from job array 20
$ scancel 20_[1-3]

# Cancel array ID 4 and 5 from job array 20
$ scancel 20_4 20_5

# Cancel all elements from job array 20
$ scancel 20

# Cancel the current job or job array element (if job array)
if [[-z $SLURM_ARRAY_JOB_ID]]; then
  scancel $SLURM_JOB_ID
else
  scancel ${SLURM_ARRAY_JOB_ID}_${SLURM_ARRAY_TASK_ID}
fi
```
## Squeue命令的使用

当一个作业阵列被提交给Slurm时，只有一条作业记录被创建。只有当工作阵列中的任务状态发生变化时，才会创建额外的工作记录，通常是当一个任务被分配资源或使用`scontrol`命令修改其状态。默认情况下，`squeue`命令将在一行中报告与单个作业记录相关的所有任务，并使用正则表达式来表示 `"array_task_id "`值，如下所示。
```
$ squeue
 JOBID     PARTITION  NAME  USER  ST  TIME  NODES NODELIST(REASON)
1080_[5-1024]  debug   tmp   mac  PD  0:00      1 (Resources)
1080_1         debug   tmp   mac   R  0:17      1 tux0
1080_2         debug   tmp   mac   R  0:16      1 tux1
1080_3         debug   tmp   mac   R  0:03      1 tux2
1080_4         debug   tmp   mac   R  0:03      1 tux3
```
`squeue`命令中还增加了`"--array "`或`"-r "`选项，可以在每行打印一个作业阵列元素，如下图所示。环境变量 `"SQUEUE_ARRAY "`等同于在`squeue`命令行中加入`"--array "`选项。
```
$ squeue -r
 JOBID PARTITION  NAME  USER  ST  TIME  NODES NODELIST(REASON)
1082_3     debug   tmp   mac  PD  0:00      1 (Resources)
1082_4     debug   tmp   mac  PD  0:00      1 (Priority)
  1080     debug   tmp   mac   R  0:17      1 tux0
  1081     debug   tmp   mac   R  0:16      1 tux1
1082_1     debug   tmp   mac   R  0:03      1 tux2
1082_2     debug   tmp   mac   R  0:03      1 tux3
```
`squeue` `--step/-s`和`--job/-j`选项可以接受相同格式的作业或步骤规范。
```
$ squeue -j 1234_2,1234_3
...
$ squeue -s 1234_2.0,1234_3.0
...
```
在`squeue`中增加了两个额外的作业输出格式字段选项：
`%F` 打印 `array_job_id` 值
`%`K打印`array_task_id`值
(所有要使用的明显的字母都已经分配给其他作业字段)。

## `Scontrol`命令的使用

使用`scontrol show job`选项可以显示两个与工作阵列支持有关的新字段。`JobID`是作业的唯一标识符。`ArrayJobID`是工作阵列中第一个元素的`JobID`。`ArrayTaskID`是这个特定条目的数组索引，可以是一个单一的数字，也可以是识别这个工作记录所代表的条目的表达式（例如，"5-1024"）。如果该工作不是工作阵列的一部分，这两个字段都不会显示。在`scontrol show job`或`scontrol show step`命令中指定的可选作业ID，可以通过指定`ArrayJobId`和`ArrayTaskId`，并在它们之间加一个下划线来识别作业阵列元素（例如：`<ArrayJobID>_<ArrayTaskId>`）。

如果指定的工作ID是`ArrayJobID`，`scontrol`命令将对工作阵列的所有元素进行操作。单个工作阵列任务可以使用`ArrayJobID_ArrayTaskID`进行修改，如下所示。
```
$ sbatch --array=1-4 -J array ./sleepme 86400
Submitted batch job 21845

$ squeue
 JOBID   PARTITION     NAME     USER  ST  TIME NODES NODELIST
 21845_1    canopo    array    david  R  0:13  1     dario
 21845_2    canopo    array    david  R  0:13  1     dario
 21845_3    canopo    array    david  R  0:13  1     dario
 21845_4    canopo    array    david  R  0:13  1     dario

$ scontrol update JobID=21845_2 name=arturo
$ squeue
 JOBID   PARTITION     NAME     USER  ST   TIME  NODES NODELIST
 21845_1    canopo    array    david  R   17:03   1    dario
 21845_2    canopo   arturo    david  R   17:03   1    dario
 21845_3    canopo    array    david  R   17:03   1    dario
 21845_4    canopo    array    david  R   17:03   1    dario
```
`scontrol`的`hold`、`holdu`、`release`、`requeue`、`requeuehold`、`suspend`和`resume`命令也可以对作业阵列的所有元素或单个元素进行操作，如下所示。
```
$ scontrol suspend 21845
$ squeue
 JOBID PARTITION      NAME     USER  ST TIME  NODES NODELIST
21845_1    canopo    array    david  S 25:12  1     dario
21845_2    canopo   arturo    david  S 25:12  1     dario
21845_3    canopo    array    david  S 25:12  1     dario
21845_4    canopo    array    david  S 25:12  1     dario
$ scontrol resume 21845
$ squeue
 JOBID PARTITION      NAME     USER  ST TIME  NODES NODELIST
21845_1    canopo    array    david  R 25:14  1     dario
21845_2    canopo   arturo    david  R 25:14  1     dario
21845_3    canopo    array    david  R 25:14  1     dario
21845_4    canopo    array    david  R 25:14  1     dario

scontrol suspend 21845_3
$ squeue
 JOBID PARTITION      NAME     USER  ST TIME  NODES NODELIST
21845_1    canopo    array    david  R 25:14  1     dario
21845_2    canopo   arturo    david  R 25:14  1     dario
21845_3    canopo    array    david  S 25:14  1     dario
21845_4    canopo    array    david  R 25:14  1     dario
scontrol resume 21845_3
$ squeue
 JOBID PARTITION      NAME     USER  ST TIME  NODES NODELIST
21845_1    canopo    array    david  R 25:14  1     dario
21845_2    canopo   arturo    david  R 25:14  1     dario
21845_3    canopo    array    david  R 25:14  1     dario
21845_4    canopo    array    david  R 25:14  1     dario
```
## 作业的依赖性

一个工作如果要依赖于整个工作阵列，应该指定自己依赖于`ArrayJobID`。由于每个数组元素可以有不同的退出代码，`afterok`和`afterernotok`条款的解释将基于作业数组中任何任务的最高退出代码。

当一个作业依赖性指定了一个作业数组的作业ID：
`after`子句在工作数组中的所有任务开始后被满足。
`afterany`子句在作业数组中的所有任务完成后得到满足。
`aftercorr`子句在指定工作中的相应任务ID成功完成后被满足（运行到完成，退出代码为0）。
`afterok`子句在作业数组中的所有任务成功完成后被满足。
`afterernotok`子句在作业数组中的所有任务完成后满足，至少有一个任务没有成功完成。

下面是使用的例子：
```
# Wait for specific job array elements
sbatch --depend=after:123_4 my.job
sbatch --depend=afterok:123_4:123_8 my.job2

# Wait for entire job array to complete
sbatch --depend=afterany:123 my.job
# Wait for corresponding job array elements
sbatch --depend=aftercorr:123 my.job

# Wait for entire job array to complete successfully
sbatch --depend=afterok:123 my.job

# Wait for entire job array to complete and at least one task fails
sbatch --depend=afternotok:123 my.job
```

## 其他命令的使用

以下Slurm命令目前不能识别作业阵列，它们的使用需要使用Slurm作业ID，每个阵列元素都是唯一的：`sbcast`, `sprio`, `sreport`, `sshare` 和 `sstat`。`sacct`、`sattach`和`strigger`命令已经被修改，允许指定作业ID或作业数组元素。修改了`sview`命令，允许显示工作的`ArrayJobId`和`ArrayTaskId`字段。如果作业不是作业阵列的一部分，这两个字段的显示值为 `"N/A"`。

## 系统管理

增加了一个新的配置参数来控制最大作业阵列的大小： `MaxArraySize`。用户可以指定的最小索引是0，最大索引是`MaxArraySize`减去1。`MaxArraySize`的默认值是1001。Slurm中支持的最大`MaxArraySize`是4000001。要注意`MaxArraySize`的值，因为作业数组为用户提供了一种快速提交大量作业的简便方法。

`sched/backfill`插件已经被修改，以提高作业阵列的性能。一旦发现作业数组中的一个元素不能运行或影响待处理作业的调度，该作业数组的其余元素将被快速跳过。

Slurm在提交作业阵列时创建了一个作业记录。额外的作业记录只有在需要时才会被创建，通常是在作业阵列的一个任务被启动时，这为管理大型作业数量提供了一个非常可扩展的机制。作业阵列的每个任务将共享相同的`ArrayJobId`，但会有他们自己独特的`ArrayTaskId`。除了`ArrayJobId`之外，每个工作都有一个独特的JobId，在任务启动时被分配。
