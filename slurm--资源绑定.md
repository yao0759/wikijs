---
title: slurm--资源绑定
description: slurm中文翻译系列，机翻后纠正了一点，发现其他错误望指出，来源：https://github.com/SchedMD/slurm/blob/master/doc/html/resource_binding.shtml
published: true
date: 2024-06-29T07:36:13.538Z
tags: slurm
editor: markdown
dateCreated: 2024-06-29T07:36:13.538Z
---



## 概述

Slurm 拥有丰富的选项集，可控制任务与资源的默认绑定。例如，任务可以绑定到单个threads、cores、sockets、NUMA 或boards。有关这些选项如何工作的更多信息，请参阅 slurm.conf 和 srun man 页面。本文重点介绍如何配置默认绑定配置。默认绑定可按per-node、per-partiton或全局设置。使用 srun **--cpu-bind** 选项指定的绑定优先级最高。如果任务分配中的任何节点有某些 CpuBind 配置参数，而任务分配中的所有其他节点要么有相同的 CpuBind 配置参数，要么没有 CpuBind 配置参数，则次高优先级绑定将是节点特定绑定。次高优先级绑定将是分区特定的 CpuBind 配置参数（如果有）。优先级最低的绑定将是 **TaskPluginParam** 配置参数指定的绑定。

1. Srun **--cpu-bind** 选项
2. node cpuBind 配置参数（如果所有节点都匹配）
3. partition CpuBind 配置参数
4. TaskPluginParam配置参数

## Srun --cpu-bind选项

srun **--cpu-bind**选项将始终用于控制任务绑定。如果**--cpu-bind**选项只包含 "verbose"（详细说明），而没有指明要绑定的实体，那么将根据 Slurm 配置参数（如下所述）使用详细说明选项和默认实体。

## 节点CpuBind配置

资源绑定信息的下一个可能来源是节点配置的 CpuBind 值，但前提是每个节点都具有相同的 CpuBind 值（或没有配置 CpuBind 值）。节点的 CpuBind 值在 slurm.conf 文件中配置。可以使用 scontrol 命令查看或修改其值。要清除节点的 CpuBind 值，请使用以下命令：

```
scontrol update NodeName=... CpuBind=off
```

如果配置了 node_features 插件，通常是为了支持将英特尔 KNL 节点启动到不同的 NUMA 和/或 MCDRAM 模式，则可以配置该插件，以便根据 NUMA 模式修改节点的 CpuBind 选项。具体方法是在 knl.conf 配置文件中指定 NumaCpuBind 参数，并将 NUMA 模式和 CpuBind 选项配对。一旦节点启动到新的 NUMA 模式，节点的 CpuBind 选项就会自动修改。例如，knl.conf 文件中的以下一行

```
NumaCpuBind=a2a=core;snc2=thread
```

在启动到 "a2a"（全对全）NUMA 模式时，将把节点的 CpuBind 字段设置为 "core"（核心）；在启动到 "snc2 NUMA "模式时，将把节点的 CpuBind 字段设置为 "thread"（线程）。任何未在 NumaCpuBind 配置文件中指定的 NUMA 模式都不会改变节点的 CpuBind 字段。

## Partition CpuBind 配置

资源绑定信息的下一个可能来源是分区配置的 CpuBind 值。partition的 CpuBind 值在 slurm.conf 文件中配置。可使用 **scontrol** 命令查看或修改其值。

## TaskPluginParam配置

资源绑定信息的最后一个可能来源是**slurm.conf**文件中的 **TaskPluginParam** 配置参数。