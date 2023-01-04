---
title: slurm--异质性工作支持
description: 
published: true
date: 2023-01-04T13:57:20.113Z
tags: slurm
editor: markdown
dateCreated: 2023-01-04T13:57:20.113Z
---

# slurm--异质性工作支持

## 概述
Slurm 17.11及以后的版本支持提交和管理异质作业的能力，其中每个组件几乎都有所有可用的作业选项，包括分区、账户和QOS（服务质量）。例如，一个作业的一部分可能需要四个核心和4GB的128个任务，而另一部分作业需要16GB的内存和一个CPU。

## 提交作业
`salloc`、`sbatch`和`srun`命令都可以用来提交异构作业。异质作业的每个组件的资源规格应该用": "字符分开。例如。

```shell
$ sbatch --cpus-per-task=4 --mem-per-cpu=1  --ntasks=128 : \
         --cpus-per-task=1 --mem-per-cpu=16 --ntasks=1 my.bash
```

为异质作业（或作业步骤）的一个组件指定的选项将被用于后续的组件，其程度预计是有帮助的。传播的选项可以根据需要为每个组件重新设置（例如，可以为每个hetjob组件指定一个不同的账户名。例如，`--immediate`和`--job-name`被传播，而`--ntasks`和`--mem-per-cpu`被重置为每个组件的默认值。下面是一个传播的选项列表。

- --account
- --acctg-freq
- --begin
- --cluster-constraint
- --clusters
- --comment
- --deadline
- --delay-boot
- --dependency
- --distribution
- --epilog (option available only in srun)
- --error
- --export
- --export-file
- --exclude
- --get-user-env
- --gid
- --hold
- --ignore-pbs
- --immediate
- --input
- --job-name
- --kill-on-bad-exit (option available only in srun)
- --label (option available only in srun)
- --mcs-label

- --mem
- --msg-timeout (option available only in srun)
- --no-allocate (option available only in srun)
- --no-requeue
- --nice
- --no-kill
- --open-mode (option available only in srun)
- --output
- --parsable
- --priority
- --profile
- --propagate
- --prolog (option available only in srun)
- --pty (option available only in srun)
- --qos
- --quiet
- --quit-on-interrupt (option available only in srun)
- --reboot
- --reservation
- --requeue
- --signal
- --slurmd-debug (option available only in srun)
- --task-epilog (option available only in srun)
- --task-prolog (option available only in srun)
- --time
- --test-only
- --time-min
- --uid
- --unbuffered (option available only in srun)
- --verbose
- --wait
- --wait-all-nodes
- --wckey
- --workdir

任务分配规范分别适用于每个作业组件。例如，考虑一个异构作业，每个组件在2个节点上被分配4个CPU。在我们的例子中，工作组件0在节点 "nid00001 "上分配2个CPU，在节点 "nid00002 "上分配2个CPU。工作组件一在节点 "nid00003 "上分配2个CPU，在节点 "nid00004 "上分配2个CPU。任务分配为 "循环"，将前4个任务循环分配到节点 "nid00001 "和 "nid00002 "上，然后将后4个任务循环分配到节点 "nid00003 "和 "nid00004 "上，如下所示。

| Node nid00001 | Node nid00002 | Node nid00003 | Node nid00004 |
| ------------- | ------------- | ------------- | ------------- |
| Rank 0        | Rank 1        | Rank 4        | Rank 5        |
| Rank 2        | Rank 3        | Rank 6        | Rank 7        |

有些选项应该只在第一个hetjob组件中指定。例如，在第二个hetjob组件的选项中指定一个批处理作业输出文件，将导致第一个hetjob组件（批处理脚本执行的地方）使用默认的输出文件名。

用于指定作业提交命令的默认选项的环境变量将被应用于异质作业的每个组件（例如`SBATCH_ACCOUNT`）。

批量作业选项可以包含在多个异质作业组件的提交脚本中。每个组件应以包含 "#SBATCH hetjob "的行来分隔，如下所示。

```shell
$ cat new.bash
#!/bin/bash
#SBATCH --cpus-per-task=4 --mem-per-cpu=16g --ntasks=1
#SBATCH hetjob
#SBATCH --cpus-per-task=2 --mem-per-cpu=1g  --ntasks=8
srun run.app

$ sbatch new.bash
```

相当于以下内容：

```shell
$ cat my.bash
#!/bin/bash
srun run.app

$ sbatch --cpus-per-task=4 --mem-per-cpu=16g --ntasks=1 : \
         --cpus-per-task=2 --mem-per-cpu=1g  --ntasks=8 my.bash
```

批量脚本将在异构作业的第一个组件的第一个节点中执行。在上面的例子中，这将是具有1个任务、4个CPU和64GB内存（4个CPU中每个有16GB）的作业组件。

如果一个异构作业被提交到不属于联盟的多个集群中运行（例如 "sbatch --cluster=alpha,beta ..."），那么整个作业将被发送到预计能够在最早时间启动所有组件的集群。

在提交异构作业时，会进行资源限制测试，以便立即拒绝那些在当前限制下无法启动的作业。异构作业的单个组件被验证，就像所有常规作业一样。异构作业作为一个整体也被测试，但在服务质量（QOS）限制方面，测试方式更为有限。就资源限制而言，异质作业的每个组件都算作一个 "作业"。

## 突发缓冲器
突发缓冲区可以是持久的，也可以与特定的作业ID相联系（至少在Cray的实现中）。由于一个异质作业由多个作业ID组成，一个特定的作业突发缓冲区将只与一个异质作业组件相关。只有一个持久的突发缓冲区可以被一个异质作业的所有组件访问。后面附有一个Cray系统的批处理脚本样本，以证明这一点。

```shell
#!/bin/bash
#SBATCH --nodes=1 --constraint=haswell
#BB create_persistent name=alpha capacity=10 access=striped type=scratch
#DW persistentdw name=alpha
#SBATCH hetjob
#SBATCH --nodes=16 --constraint=knl
#DW persistentdw name=alpha
...
```

注意：Cray的DataWarp接口直接读取作业脚本，但不知道 "Slurm的 "hetjob "指令，所以Slurm在内部为每个作业组件重建脚本，以便只有该作业组件的突发缓冲指令被包含在该脚本中。批处理脚本的第一个作业组件将被修改，以便将其他作业组件的 "#DW "指令替换为 "#EXCLUDED DW"，以防止它们被Cray基础设施解释。由于批处理脚本将只由第一个作业组件执行，随后的作业组件将不包括原始脚本的命令。这些脚本是由Slurm为内部目的而建立和管理的（并可从各种Slurm命令中看到），如上图所示的用户脚本。下面是一个例子。

重建的第一个工作组件的脚本

```shell
#!/bin/bash
#SBATCH --nodes=1 --constraint=haswell
#BB create_persistent name=alpha capacity=10 access=striped type=scratch
#DW persistentdw name=alpha
#SBATCH hetjob
#SBATCH --nodes=16 --constraint=knl
#EXCLUDED DW persistentdw name=alpha
...
```

重建第二个工作组件的脚本

```shell
#!/bin/bash
#SBATCH --nodes=16 --constraint=knl
#DW persistentdw name=alpha
exit 0
```

## 管理作业

在Slurm中为一个异质作业维护的信息包括。

- job_id：一个异质作业的每个组件都有自己独特的job_id。
- het_job_id：这个标识号适用于异质作业的所有组件。同一作业的所有组件将有相同的 het_job_id 值，它将等于第一个组件的 job_id。我们把这称为 "异质作业领导"。
- het_job_id_set：正则表达式，识别与工作相关的所有job_id值。
- het_job_offset：适用于异质作业的每个组件的唯一序列号。第一个组件的 het_job_offset 值为 0，第二个组件的值为 1，等等。

| job_id | het_job_id | het_job_offset | het_job_id_set |
| ------ | ---------- | -------------- | -------------- |
| 123    | 123        | 0              | 123-127        |
| 124    | 123        | 1              | 123-127        |
| 125    | 123        | 2              | 123-127        |
| 126    | 123        | 3              | 123-127        |
| 127    | 123        | 4              | 123-127        |

表1：工作ID的例子

`squeue`和`sview`命令使用`"<het_job_id>+<het_job_offset>"`的格式报告一个异质作业的组成部分。例如，"123+4 "代表异质作业ID 123和它的第五个组件（注意：第一个组件的`het_job_offset`值为0）。

对一个特定的工作ID的请求，确定一个异质工作的第一个组件的ID（即 "异质工作领导"），将返回该工作的所有组件的信息。例如。

```
$ squeue --job=93
JOBID PARTITION  NAME  USER ST  TIME  NODES NODELIST
 93+0     debug  bash  adam  R 18:18      1 nid00001
 93+1     debug  bash  adam  R 18:18      1 nid00011
 93+2     debug  bash  adam  R 18:18      1 nid00021
```

取消或以其他方式向异质作业领导发出信号的请求，将适用于该异质作业的所有组件。使用 "#+#"符号取消异构作业的特定组件的请求将仅适用于该特定组件。例如。

```shell
$ squeue --job=93
JOBID PARTITION  NAME  USER ST  TIME  NODES NODELIST
 93+0     debug  bash  adam  R 19:18      1 nid00001
 93+1     debug  bash  adam  R 19:18      1 nid00011
 93+2     debug  bash  adam  R 19:18      1 nid00021
$ scancel 93+1
$ squeue --job=93
JOBID PARTITION  NAME  USER ST  TIME  NODES NODELIST
 93+0     debug  bash  adam  R 19:38      1 nid00001
 93+2     debug  bash  adam  R 19:38      1 nid00021
$ scancel 93
$ squeue --job=93
JOBID PARTITION  NAME  USER ST  TIME  NODES NODELIST
```

当一个异质作业处于挂起状态时，只有整个作业可以被取消，而不是其单个组件。取消处于待定状态的异构作业的单个组件的请求将返回一个错误。在作业开始执行后，单个组件可以被取消。

工作状态变化的电子邮件通知（`--mail-type`选项）只支持异构工作领导。对异构作业的其他组件的电子邮件通知请求将被默默地忽略。

使用 scontrol 命令修改作业的单个组件的请求必须用 "#+#"符号指定作业 ID。通过指定 het_job_id 来修改作业的请求将修改异质作业的所有组件。例如。

```
# Change the account of component 2 of heterogeneous job 123:
$ scontrol update jobid=123+2 account=abc

# Change the time limit of all components of heterogeneous job 123:
$ scontrol update jobid=123 timelimit=60
```

执行以下操作的请求只能针对一个异质作业领导，并将应用于该异质作业的所有组件。对异构的单个组件进行操作的请求将返回一个错误。

- requeue
- resume
- suspend

`sbcast`命令支持异质的作业分配。默认情况下，`sbcast`将复制文件到作业分配中的所有节点。可以使用`-j/--jobid`选项来复制文件到各个组件，如下图所示。

```shell
$ sbcast --jobid=123   data /tmp/data
$ sbcast --jobid=123.0 app0 /tmp/app0
$ sbcast --jobid=123.1 app1 /tmp/app1
```

`srun`命令的`--bcast`选项将把文件传输到与`--het-group`选项所指定的要启动的应用程序相关的节点。

Slurm有一个配置选项来控制一些命令对异质作业的行为。默认情况下，取消、保留或释放一个不是het_job_id的作业ID，而是一个作业组件的ID的请求将只操作异构作业的那个组件。如果`SchedulerParameters`配置参数包括 "whole_hetjob "选项，那么如果任何作业组件被指定为操作对象，该操作将适用于该作业的所有组件。在下面的例子中，如果配置了`SchedulerParameters=whole_hetjob`，`scancel`命令将取消作业93的所有组件，否则只有作业93+1将被取消。如果指定了一个特定的异质作业组件（例如：`"scancel 93+1"`），那么只有那个组件会被影响。

```shell
$ squeue --job=93
JOBID PARTITION  NAME  USER ST  TIME  NODES NODELIST
 93+0     debug  bash  adam  R 19:18      1 nid00001
 93+1     debug  bash  adam  R 19:18      1 nid00011
 93+2     debug  bash  adam  R 19:18      1 nid00021
$ scancel 94 (where job ID 94 is equivalent to 93+1)
# Cancel 93+0, 93+1 and 93+2 if SchedulerParameters includes "whole_hetjob"
# Cancel only 93+1 if SchedulerParameters does not include "whole_hetjob"
```

### accounting

Slurm 的会计数据库记录了 `het_job_id `和 `het_job_offset` 字段。`sacct`命令使用`"<het_job_id>+<het_job_offset>"`的格式报告工作，并且可以接受工作ID规格，使用相同格式进行过滤。如果het_job_id值被指定为作业过滤器，那么该作业的所有组成部分的信息将被默认报告，如下图所示。`--whole-hetjob=[yes|no]`选项可以用来强制报告该作业的所有组件的信息，或者只报告所要求的特定组件的信息，不管作业过滤器是否包括het_job_id（领导）。

```shell
$ sacct -j 67767
  JobID JobName Partition Account AllocCPUS     State ExitCode 
------- ------- --------- ------- --------- --------- -------- 
67767+0     foo     debug    test         2 COMPLETED      0:0 
67767+1     foo     debug    test         4 COMPLETED      0:0 

$  sacct -j 67767+1
  JobID JobName Partition Account AllocCPUS     State ExitCode 
------- ------- --------- ------- --------- --------- -------- 
67767+1     foo     debug    test         4 COMPLETED      0:0 

$  sacct -j 67767 --whole-hetjob=no
  JobID JobName Partition Account AllocCPUS     State ExitCode
------- ------- --------- ------- --------- --------- --------
67767+0     foo     debug    test         4 COMPLETED      0:0

$ sacct -j 67767+1 --whole-hetjob=yes
  JobID JobName Partition Account AllocCPUS     State ExitCode
------- ------- --------- ------- --------- --------- --------
67767+0     foo     debug    test         2 COMPLETED      0:0
67767+1     foo     debug    test         4 COMPLETED      0:0
```

## 启动应用程序（工作步骤）
`srun命令是用来启动应用程序的。默认情况下，应用程序只在异构作业的第一个组件上启动，但有一些选项可以支持不同的行为。

srun的`"--het-group "`选项定义了哪些hetjob组件要为它们启动应用程序。`--het-group`选项采用一个表达式，定义哪些组件将为单独执行`srun`命令而启动一个应用程序。该表达式可以包含一个或多个以逗号分隔的组件索引值。索引值的范围可以用连字符分隔的列表来指定。默认情况下，一个应用程序只在组件编号为0时启动。下面是一些例子。

- --het-group=2
- --het-group=0,4
- --het-group=1,3-5

**重要提示：**在一个以上的作业分配中执行一个应用程序的能力并不适用所有的MPI实现或Slurm MPI插件。可以通过在Slurm的`SchedulerParameters`配置参数中添加 `"disable_hetjob_steps "`来禁用Slurm在整个集群中执行这种应用程序的能力。

**重要提示：**虽然`srun`命令可以用来启动异构作业步骤，但mpirun需要大量的修改来支持异构应用。我们知道目前还没有这样的mpirun开发工作。

默认情况下，由单个srun命令的执行所启动的应用程序（即使是异构作业的不同组件）被合并到一个`MPI_COMM_WORLD`中，具有不重叠的任务ID。

与`salloc`和`sbatch`命令一样，": "字符被用来分隔异构作业的多个组件。这一惯例意味着独立的": "字符不能被用作由`srun`启动的应用程序的参数。这包括为每个作业组件执行不同的应用程序和参数的能力。如果某些异质作业组件缺乏应用程序规范，那么提供的下一个应用程序规范将用于缺乏规范的早期组件，如下所示。

```
$ srun --label -n2 : -n1 hostname
0: nid00012
1: nid00012
2: nid00013
```

如果多个`srun`命令同时执行，这可能会导致资源争夺（例如，由于两个`srun`命令同时执行，内存限制使一些作业步骤组件无法被分配资源）。如果`srun --het-group`选项被用来创建多个作业步骤（用于异质作业的不同组件），这些作业步骤将被按顺序创建。当多个`srun`命令同时执行时，这可能会导致一些步骤的分配发生，而其他步骤被延迟。只有在所有工作步骤的分配被授予后，应用程序才会被启动。

一个工作步骤的所有组件将有相同的步骤ID值。如果工作步骤是在工作组件的子集上启动，个别工作组件的步骤ID值可能会有差距。

```
$ salloc -n1 : -n2 beta bash
salloc: Pending job allocation 1721
salloc: Granted job allocation 1721
$ srun --het-group=0,1 true   # Launches steps 1721.0 and 1722.0
$ srun --het-group=0   true   # Launches step  1721.1, no 1722.1
$ srun --het-group=0,1 true   # Launches steps 1721.2 and 1722.2
```

工作步骤分配中指定的最大het-group（明确指定或由": "分隔符暗示）不得超过异质工作分配中的组件数量。例如

```
$ salloc -n1 -C alpha : -n2 -C beta bash
salloc: Pending job allocation 1728
salloc: Granted job allocation 1728
$ srun --het-group=0,1 hostname
nid00001
nid00008
nid00008
$ srun hostname : date : id
error: Attempt to run a job step with het-group value of 2,
       but the job allocation has maximum value of 1
```

## 环境变量

Slurm环境变量将为作业的每个组件独立设置，在通常的名称后加上`"_HET_GROUP_"`和一个序列号。此外，`"SLURM_JOB_ID "`环境变量将包含异质作业领导者的作业ID，`"SLURM_HET_SIZE "`将包含作业中的组件数量。注意，如果使用`srun`与一个特定的het组（例如 `--het-group=1`），`"SLURM_JOB_ID "`将包含异质作业领导者的作业ID。特定异构组件的作业ID在 `"SLURM_JOB_ID_HET_GROUP_<component_id>"`中设置。比如说。

```shell
$ salloc -N1 : -N2 bash
salloc: Pending job allocation 11741
salloc: job 11741 queued and waiting for resources
salloc: job 11741 has been allocated resources
$ env | grep SLURM
SLURM_JOB_ID=11741
SLURM_HET_SIZE=2
SLURM_JOB_ID_HET_GROUP_0=11741
SLURM_JOB_ID_HET_GROUP_1=11742
SLURM_JOB_NODES_HET_GROUP_0=1
SLURM_JOB_NODES_HET_GROUP_1=2
SLURM_JOB_NODELIST_HET_GROUP_0=nid00001
SLURM_JOB_NODELIST_HET_GROUP_1=nid[00011-00012]
...
$ srun --het-group=1 printenv SLURM_JOB_ID
11741
11741
$ srun --het-group=0 printenv SLURM_JOB_ID
11741
$ srun --het-group=1 printenv SLURM_JOB_ID_HET_GROUP_1
11742
11742
$ srun --het-group=0 printenv SLURM_JOB_ID_HET_GROUP_0
11741
```

各种MPI实现在很大程度上依赖于Slurm环境变量的正常运行。在单个`MPI_COMM_WORLD`中执行的单个MPI应用程序需要一组统一的环境变量来反映单个作业分配。下面的例子显示了Slurm如何为MPI设置环境变量。

```
$ salloc -N1 : -N2 bash
salloc: Pending job allocation 11741
salloc: job 11751 queued and waiting for resources
salloc: job 11751 has been allocated resources
$ env | grep SLURM
SLURM_JOB_ID=11751
SLURM_HET_SIZE=2
SLURM_JOB_ID_HET_GROUP_0=11751
SLURM_JOB_ID_HET_GROUP_1=11752
SLURM_JOB_NODELIST_HET_GROUP_0=nid00001
SLURM_JOB_NODELIST_HET_GROUP_1=nid[00011-00012]
...
$ srun --het-group=0,1 env | grep SLURM
SLURM_JOB_ID=11751
SLURM_JOB_NODELIST=nid[00001,00011-00012]
...
```

## 例子

创建一个异质资源分配，包含一个具有256GB内存和 "haswell "特性的节点，加上32个具有 "knl "特性的节点上的2176个内核。然后在 "haswell "节点上启动一个名为 "server"的程序，在 "knl "节点上启动 "client"。每个程序都将在自己的`MPI_COMM_WORLD`中。

```
salloc -N1 --mem=256GB -C haswell : \
       -n2176 -N32 --ntasks-per-core=1 -C knl bash
srun server &
srun --het-group=1 client &
wait
```

上述例子的这个变体在一个`MPI_COMM_WORLD`中启动了程序 "server "和 "client"。

```
salloc -N1 --mem=256GB -C haswell : \
       -n2176 -N32 --ntasks-per-core=1 -C knl bash
srun server : client
```

`SLURM_PROCID`环境变量将被设置为反映一个全局任务等级。每个产生的进程将有一个唯一的`SLURM_PROCID`。

同样，`SLURM_NPROCS`和`SLURM_NTASKS`环境变量将被设置为反映全局任务数（两个环境变量将有相同的值）。`SLURM_NTASKS将被设置为所有组件中任务的总计数。注意，任务等级和计数值是MPI需要的，通常是通过检查Slurm环境变量来确定的。

## 限制

回填调度器在未来如何跟踪CPU和内存的使用方面有限制。这通常要求回填调度器能够将异构作业的每个组件分配到不同的节点上，以便开始其资源分配，即使作业的多个组件确实在同一节点上得到了资源分配。

在集群联盟中，异构作业将完全在提交该作业的集群上执行。异构作业将没有资格在集群之间迁移，也没有资格让作业的不同组件在联盟的不同集群上执行。

在提交请求多个重叠分区的异构作业时，必须谨慎行事。当分区共享相同的资源时，有可能通过让第一个作业组件请求足够多的节点而使调度器无法满足随后的请求，从而使你自己的作业陷入饥饿。考虑一个例子，你有包含10个节点的分区`p1`和存在于5个相同节点的分区`p2`。如果你提交了一个异构作业，要求`p1`中有5个节点，`p2`中有5个节点，调度器可能会尝试从`p2`分区中分配一些节点给第一个作业组件，阻止调度器完成第二个请求，导致作业永远无法启动。

不支持异质作业的作业阵列。

`srun`命令的`--no-allocate`选项不支持异构作业。

每个异构作业组件只能由单个` srun` 命令启动一个作业步骤（例如，`"srun --het-group=0 alpha : --het-group=0 beta "`不被支持）。

`sattach` 命令一次只能用于附加到异构作业的单个组件。

许可证请求仅在第一个组件作业上正确工作（例如，`"sbatch -L ansys:2 : script.sh"`）。

异构作业仅由回填调度器插件进行调度。更经常执行的调度逻辑只在先入先出（FIFO）的基础上启动作业，缺乏对异质作业的所有组件进行并发调度的逻辑。

在GANG调度操作上不支持异质作业。

Cray ALPS系统不支持异构作业。

IBM PE系统不支持异构作业。

Slurm的PERL APIs目前不支持异质作业。

`srun --multi-prog`选项不能用于跨越一个以上的异构作业组件。

`srun --open-mode`选项默认设置为 "append"。

古代版本的OpenMPI和它们的衍生物（即Cray MPI）依赖于Slurm分配给它们的通信端口。如果一个异构作业步骤的任何组件无法获得分配的端口，这样的MPI作业将经历步骤启动失败。非异构作业步骤将使用一组新的通信端口重新尝试启动步骤（Slurm行为没有改变）。

## 异构步骤

Slurm 20.11版引入了从非同构作业分配中请求异构作业步骤的能力。这使你可以灵活地对作业步骤进行不同的布局，而不需要使用异质作业，在这种情况下，为组件设置单独的作业可能是不可取的。

异质步骤的一些限制是，这些步骤必须能够在独特的节点上运行。你也不能从一个异构作业中请求异构步骤。

一个例子是，如果你有一个任务需要在每个处理器上使用1个GPU，而另一个任务需要在一个只有一个处理器的节点上使用所有可用的GPU。这可以像这样完成。

```shell
$ salloc -N2 --exclusive --gpus=10
salloc: Granted job allocation 61034
$ srun -N1 -n4 --gpus=4 printenv SLURMD_NODENAME : -N1 -n1 --gpus=6 printenv SLURMD_NODENAME
node02
node01
node01
node01
node01
```

## 系统管理员信息
作业提交插件对异质作业的每个组件都是独立调用的。

`spank_init_post_opt()`函数为异质作业的每个组件调用一次。这允许在每个作业组件的基础上定义站点选项。

异质作业的调度仅由`sched/backfill`插件执行，所有的异质作业组件要么在同一时间被调度，要么被推迟。在回填评估之前，异质作业的待定原因不会被设置。为了确保异构和非异构作业的及时启动，回填调度器在每个迭代中交替使用两种不同的模式。在第一种模式下，如果一个异质作业组件不能立即启动，它的预期启动时间将被记录下来，该作业的所有后续组件将被考虑在不早于最新组件的预期启动时间的情况下启动。在第二种模式下，所有异构作业组件将被考虑在不早于最新组件的预期启动时间的情况下启动。在第二种模式完成后，所有异构作业的预期开始时间数据被清除，第一种模式将在下一次回填调度器迭代中使用。常规（非异构作业）在回填调度器的每次迭代中都是独立调度的。

例如，考虑一个有三个组件的异构作业。当被认为是独立作业时，这些组件可以在现在（组件0）、现在加2小时（组件1）和现在加1小时（组件2）的时间启动。当回填调度器以第一种模式运行时。

- 组分0将被注意到现在可以启动，但由于要启动额外的组分而没有启动。

- 组成1将被注意到可能在2小时内启动。
- 组件2在未来2小时内不会被考虑调度，这就留下了一些额外的资源可供调度给其他工作。

当回填调度器接下来执行时，它将使用第二种模式，并且（假设没有其他状态变化）所有三个作业组件都将被认为可用于调度，时间不早于未来2小时，这可能允许其他作业在异质作业组件0启动之前被分配资源。

在下一次迭代中使用第一种模式之前，异质作业开始时间数据将被清除，以便考虑系统状态的变化，这可能允许异质作业在比先前确定的更早的时间启动。

在提交异质作业时，会进行资源限制测试，以便立即拒绝那些在当前限制下无法启动的作业。异质作业的各个组成部分被验证，就像所有常规作业一样。异构作业作为一个整体也被测试，但在服务质量（QOS）限制方面，测试的方式更为有限。这是由于每个作业组件有多达三套限制（关联、作业QOS和分区QOS）的复杂性。请注意，成功提交任何作业（异质或其他）并不能确保作业能够在不超过某些限制的情况下启动。例如，一个作业的CPU限制测试没有考虑到CPU可能不是单独分配的，而是按整个核心、套接字或节点进行资源分配的。就资源限制而言，一个异构作业的每个组件都算作一个 "作业"。

例如，一个用户可能有2个并发运行作业的限制，并提交一个有3个组件的异构作业。这种情况会对其他作业的调度产生不利影响，特别是其他异构作业。