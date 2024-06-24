---
title: slurm--可追踪资源TRES
description: slurm中文翻译系列，机翻后纠正了一点，发现其他错误望指出，来源：https://github.com/SchedMD/slurm/blob/master/doc/html/tres.shtml
published: true
date: 2024-06-24T04:02:22.737Z
tags: slurm
editor: markdown
dateCreated: 2023-03-19T15:31:02.410Z
---


TRES是一种可以跟踪使用的资源，或用于执行限制。TRES是一个类型和一个名称的组合。类型是预定义的。目前的TRES类型有：

- BB (burst buffers)
- Billing
- CPU
- Energy
- FS（文件系统）
- GRES
- IC (互连)
- License
- Mem (内存)
- Node
- Pages
- VMem (虚拟内存/大小)

计费TRES是根据分区的`TRESBillingWeights`计算的。虽然分区上的TRES权重可以定义为双数，但工作的计费TRES值是以整数存储的。在计算一项工作的公平份额时，情况并非如此，因为该值被视为双数。

有效的`'FS'`TRES 是 `'disk'`（本地磁盘）和 `'lustre'`。这些主要是为了报告使用情况，而不是限制访问。

有效的`'IC'`TRES是`'OFED'`。这些主要是为了报告使用情况，而不是限制访问。

## `slurm.conf`设置

- AccountingStorageTRES

  用于定义哪些TRES要在系统上被追踪。默认情况下，Billing、CPU、Energy、Memory、Node、FS/disk、Pages和VMem被跟踪。这些默认的TRES不能被禁用，只能被附加到。下面的例子。

  ```
  AccountingStorageTRES=gres/craynetwork,license/iop1,bb/cray
  ```

  将跟踪billing, cpu, energy, memory, nodes, fs/disk, pages and vmem，以及一个叫做craynetwork的GRES和一个叫做iop1的许可证。它还将跟踪Cray突发缓冲器的使用情况。每当这些资源在集群上被使用，它们就会被记录下来。TRES是在slurmctld启动时自动在数据库中设置的。

  需要关联名称的TRES是BB、GRES和License。从上面的例子中可以看出，GRES和License在每个系统上通常是不同的。BB TRES的名字与正在使用的突发缓冲区插件相同。在上面的例子中，我们使用的是Cray的突发缓冲器插件。

  当包括一个特定的GRES的子类型时，也建议包括它的通用类型，否则只有通用类型的请求不会被计算在内。例如，如果我们想核算`gres/gpu:tesla`，我们也会在类似`srun --gres=gpu:1`的请求中包括`gres/gpu`来核算gpus。

  ```
  AccountingStorageTRES=gres/gpu,gres/gpu:tesla
  ```

- PriorityWeightTRES
  以逗号分隔的TRES类型和权重列表，设定每个TRES类型对作业优先级的贡献程度。

  ```
  PriorityWeightTRES=CPU=1000,Mem=2000,GRES/gpu=3000
  ```

  仅当`PriorityType=priority/multifactor`和`AccountingStorageTRES`被配置到每个TRES类型时适用。默认值为0。

  计费TRES不能用于优先级计算，因为这个数字在作业被分配资源后才会产生--因为不同分区的数字会发生变化。

- TRESBillingWeights

  对于每个分区，这个选项用来定义每个 TRES 类型的计费权重，用于计算作业的使用量。

  计费权重是以逗号分隔的 `TRES=Weight` 对列表的形式指定的。

  任何 TRES 类型都可用于计费。注意，内存和突发缓冲区的基本单位是兆字节。

  默认情况下，TRES的计费被计算为所有TRES类型的总和乘以其相应的计费权重。

  一个资源的加权量可以通过在计费权重后面添加K、M、G、T或P的后缀来调整。例如，在一个分配了8GB的作业上，内存权重为 "mem=.25"，将被计费2048（8192MB*.25）个单位。同一作业的内存权重 "mem=.25G "将被计费2（8192MB*（.25/1024））个单位。

  当一个作业被分配到1个CPU和8GB内存的分区上，配置为：。

  ```
  TRESBillingWeights="CPU=1.0,Mem=0.25G,GRES/gpu=2.0"
  ```

  可计费的TRES将是。

  ```
  (1*1.0) + (8*0.25) + (0*2.0) = 3.0
  ```

  如果配置了`PriorityFlags=MAX_TRES`，计费TRES被计算为一个节点上单个TRES的最大值（如cpu，mem，gres）加上所有全局TRES的总和（如licenses）。使用上述相同的例子，可计费的TRES将是。

  ```
  max(1*1.0, 8*0.25) + (0*2.0) = 2.0
  ```

  如果没有定义`TRESBillingWeights`，那么作业将按照分配的CPU总数计费。

  **注意：** `TRESBillingWeights`只在计算公平份额时使用，并不直接影响作业的优先级，因为它目前不用于作业的大小。如果你希望TRES'在作业的优先级中发挥作用，那么请参考`PriorityWeightTRES`选项。

  **注意：** 与`PriorityWeightTRES`一样，只有在`AccountingStorageTRES`中定义的TRES可用于`TRESBillingWeights`。

  **注意：** 工作可以根据计算的TRES计费值进行限制。更多信息请参见资源限制文件。

  **注意：** 如果一个计费TRES被定义为一个权重，它将被忽略。

## sacct
`sacct`可以通过在`--format`选项中添加 `"TRES"`来查看每个作业的TRES。

## sacctmgr
`sacctmgr`用来查看系统中全局可用的各种TRES。 `sacctmgr show tres`可以做到这一点。

## sreport
`sreport`报告不同的TRES。只要使用逗号分隔的输入选项 `--tres=` 就可以让`sreport`生成所要求的TRES类型的报告。关于这些报告的更多信息可以在sreport手册中找到。

在`sreport`中，`"Reported"`计费TRES是由每个节点的最大计费TRES乘以时间范围计算出来的。例如，如果一个节点是多个分区的一部分，并且每个分区有不同的`TRESBillingWeights`定义，则该节点的Billing TRES将是分区中最高的。如果一个节点的任何分区没有定义`TRESBillingWeights`，那么计费TRES将等于该节点的CPU数量。