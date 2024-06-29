---
title: slurm--qos
description: slurm中文翻译系列，机翻后纠正了一点，发现其他错误望指出，来源：https://github.com/SchedMD/slurm/blob/master/doc/html/qos.shtml
published: true
date: 2024-06-29T14:45:05.069Z
tags: slurm
editor: markdown
dateCreated: 2024-06-29T14:45:05.069Z
---


人们可以为提交给Slurm的每个作业指定一个服务质量（QOS）。与作业相关的服务质量将以三种方式影响作业：

- 作业调度的优先级
- 作业抢占
- 作业限制
- 分区QOS
- 其他QOS选项
- 配置
- 例子

QOS是使用`sacctmgr`工具在Slurm数据库中定义的。

工作使用`sbatch`、`salloc`和`srun`命令的"`--qos=`"选项请求一个QOS。

## 作业调度的优先级 

工作调度的优先权是由一些因素组成的，如`priority/multifactor`插件中所述。其中一个因素是QOS优先级。每个QOS都在Slurm数据库中定义，包括一个相关的优先级。请求并允许一个QOS的作业将在作业的多因素优先级计算中纳入与该QOS相关的优先级。

要启用`priority/multifactor`计算中的QOS优先级部分，必须在`slurm.conf`文件中定义 "`PriorityWeightQOS`"配置参数，并分配一个大于0的整数值。

一个作业的QOS只在多因素插件加载时影响其调度优先级。

## 作业优先权 

Slurm为排队作业提供了两种方法来抢占正在运行的作业，释放正在运行的作业的资源并将其分配给排队的作业。详情请看抢占描述。

抢占方式由`slurm.conf`中定义的 "`PreemptType`"配置参数决定。当 "`PreemptType`"被设置为 "`preempt/qos`"时，一个排队作业的QOS将被用来决定它是否可以抢占一个正在运行的作业。需要注意的是，用于确定作业是否有资格被抢占的QOS是与该作业相关的QOS，而不是分区QOS。

QOS 可以被分配（使用 `sacctmgr`）一个它可以抢占的其他 QOS 的列表。当有一个排队的作业的QOS被允许抢占另一个QOS的运行作业时，Slurm调度器将抢占运行的作业。

QOS选项`PreemptExemptTime`指定了作业被考虑抢占前的最小运行时间。该QOS选项优先于同名的全局选项。带有`PreemptExemptTime`的分区QOS优先于带有`PreemptExemptTime`的作业QOS，除非该作业QOS启用了OverPartQOS标志。

## 工作限制

每个QOS都被分配了一组限制，这些限制将被应用到作业中。这些限制反映了Slurm数据库中定义的用户/账户/群集/分区关联的限制，并在资源限制部分进行了描述。当一个QOS的限制被定义后，它们将优先于协会的限制。

## 分区QOS 

一个QOS可以被附加到一个分区。这意味着一个分区将拥有与QOS相同的限制。这也提供了真正的 "浮动 "分区的能力，这意味着如果你把所有的节点分配给一个分区，然后在分区的 QOS 中限制 GrpCPUs 或 GrpNodes 的数量，那么这个分区将可以访问所有的节点，但只能在其中的资源数量上运行。

分区的QOS将覆盖作业的QOS。如果需要相反的情况，你需要让作业的QOS有'`OverPartQOS`'标志，这将颠倒优先顺序。

## 其他QOS选项 
- **Flags** slurmctld用来覆盖或强制执行某些特性。有效的选项有：

  - **DenyOnLimit** 如果设置了，使用这个QOS的作业在提交时将被拒绝，如果它们不符合作为独立作业的QOS'Max'限制。如果作业在考虑其他作业时超过了这些限制，但在单独考虑时符合限制，则不会被拒绝。相反，它们将被搁置，直到有资源可用（如默认情况下没有`DenyOnLimit`）。组限制（如GrpTRES）也将被视为 "最大 "限制（如`MaxTRESPerNode`），如果作业作为独立的作业会违反限制，将被拒绝。目前这只适用于QOS和协会限制。
  - **EnforceUsageThreshold** 如果设置，并且QOS也有一个UsageThreshold，任何用这个QOS提交的作业如果低于`UsageThreshold`将被保留，直到他们的Fairshare Usage超过Threshold。
  - **NoReserve** 如果这个标志被设置并且使用回填调度，使用这个QOS的作业将不会在回填调度的通过时间分配的资源地图中保留资源。这个标志用于可能被与所有其他QOS相关的作业抢占的QOS（例如，与 "待机 "QOS一起使用）。如果这个标志与一个不能被所有其他QOS抢占的QOS一起使用，可能会导致较大的作业饿死。
  - **PartitionMaxNodes** 如果设置，使用此QOS的作业将能够覆盖所请求的分区的MaxNodes限制。
  - **PartitionMinNodes** 如果设置，使用此QOS的作业将能够覆盖所请求的分区的MinNodes限制。
  - **OverPartQOS** 如果设置，使用此 QOS 的作业将能够覆盖所请求分区的 QOS 限制所使用的任何限制。
  - **PartitionTimeLimit** 如果设置，使用此 QOS 的作业将能够覆盖所请求分区的 TimeLimit。
  - **RequiresReservation** 如果设置，使用此QOS的作业在提交作业时必须指定一个保留。这个选项在限制QOS的使用方面很有用，因为QOS可能有更大的抢占能力或额外的资源，只允许在预订范围内使用。
  - **NoDecay** 如果设置了这个选项，这个QOS的GrpTRESMins、GrpWall和UsageRaw就不会被`slurm.conf`的`PriorityDecayHalfLife`或`PriorityUsageResetPeriod`设置所衰减。这允许QOS提供总量限制，一旦被消耗，将不会自动补充。这样的QOS将作为一个有时间限制的资源配额，供访问它的协会使用。对于使用QOS的协会来说，账户/用户的使用量仍将被衰减。可以增加`QOS GrpTRESMins`和`GrpWall`限制，或将`QOS RawUsage`值重置为0（零），以再次允许用该QOS提交的作业运行（如果以`QOSGrp{TRES}MinutesLimit`或`QOSGrpWallLimit`为理由悬而未决，其中`{TRES}`是某种类型的可追踪资源）。

  - **UsageFactorSafe** 如果设置了，并且`AccountingStorageEnforce`包括Safe，作业将只能在应用UsageFactor的情况下运行到完成。

- **GraceTime** 赎回宽限时间 延长到已被选择为赎回的作业。

- **UsageFactor** 在作业的TRES使用量（如RawUsage、TRESMins、TRESRunMins）中考虑的一个浮点数。例如，如果usagefactor是2，作业每运行一秒钟TRESBillingUnit，它就会被算作2。如果usagefactor是0.5，每一秒将只计算一半的时间。设置为0将不增加作业的定时使用。

  使用系数只适用于作业的QOS，而不是分区的QOS。

  如果UsageFactorSafe标志被设置，并且AccountingStorageEnforce包括Safe，作业将只能在应用了UsageFactor的情况下运行到完成时才能运行。

  如果没有设置UsageFactorSafe标志，并且AccountingStorageEnforce包括Safe，作业将能够在没有应用UsageFactor的情况下被安排，并且能够运行而不会因为限制而被杀死。

  如果没有设置UsageFactorSafe标志，并且AccountingStorageEnforce不包括Safe，作业将能够在没有应用UsageFactor的情况下被安排，并且可能由于限制而被杀死。

  参见slurm.conf手册中的AccountingStorageEnforce。

  默认值为1。要清除先前设置的值，请使用修改命令，新值为-1。

- **UsageThreshold** 一个浮动值，代表允许运行作业的协会的最低公平份额。如果一个协会低于这个阈值，并且有待处理的工作或提交新的工作，这些工作将被保留，直到用量回到阈值以上。使用sshare来查看系统上的当前份额。

## 配置

综上所述，QOS和它们相关的限制是在Slurm数据库中使用sacctmgr工具定义的。只有当多因素优先级插件被加载并且在`slurm.conf`文件中定义了非零的 "`PriorityWeightQOS`"时，QOS才会影响作业调度的优先级。只有当`slurm.conf`文件中的 "`PreemptType`"被定义为 "preempt/qos "时，QOS才会决定作业的抢占。为 QOS 定义的限制（以及上文所述）将覆盖user/account/cluster/partition关联的限制。

## QOS 示例

QOS操作的例子。所有的QOS操作都是通过`sacctmgr`命令完成的。考虑到大量的限制和选项，'`sacctmgr show qos'`的默认输出非常长，所以最好使用格式选项来过滤显示。

默认情况下，当一个集群被添加到数据库中时，会创建一个名为normal的默认qos。

```
$ sacctmgr show qos format=name,priority
      Name   Priority
---------- ----------
    normal          0
```

添加新的qos

```bash
$ sacctmgr add qos zebra
 Adding QOS(s)
  zebra
 Settings
  Description    = QOS Name

$ sacctmgr show qos format=name,priority
      Name   Priority
---------- ----------
    normal          0
     zebra          0
```

配置qos优先级

```bash
$ sacctmgr modify qos zebra set priority=10
 Modified qos...
  zebra

$ sacctmgr show qos format=name,priority
      Name   Priority
---------- ----------
    normal          0
     zebra         10
```

配置其他限制

```bash
$ sacctmgr modify qos zebra set GrpCPUs=24
 Modified qos...
  zebra

$ sacctmgr show qos format=name,priority,GrpCPUs
      Name   Priority  GrpCPUs
---------- ---------- --------
    normal          0
     zebra         10       24
```

添加qos到一个用户账号

```bash
$ sacctmgr modify user crock set qos=zebra

$ sacctmgr show assoc format=cluster,user,qos
   Cluster       User                  QOS
---------- ---------- --------------------
canis_major                          normal
canis_major      root                normal
canis_major                          normal
canis_major     crock                zebra
```

用户能够属于多个qos

```bash
$ sacctmgr modify user crock set qos+=alligator
$ sacctmgr show assoc format=cluster,user,qos
   Cluster       User                  QOS
---------- ---------- --------------------
canis_major                          normal
canis_major      root                normal
canis_major                          normal
canis_major     crock       alligator,zebra
```

删除qos

```bash
$ sacctmgr delete qos alligator
 Deleting QOS(s)...
  alligator
```