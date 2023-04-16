---
title: 无配置slurm
description: https://github.com/SchedMD/slurm/blob/master/doc/html/configless_slurm.shtml
published: true
date: 2023-04-16T12:57:57.959Z
tags: slurm
editor: markdown
dateCreated: 2023-04-16T12:57:57.959Z
---

## "Configless"Slurm

"Configless"Slurm是一个允许计算节点--特别是slurmd进程--和在登录节点上运行的用户命令直接从slurmctld中获取配置信息，而不是从预先分布的本地文件中获取。你的集群确实需要在Slurm控制器上有一套中央配置文件--在Slurm的说法中，"无配置 "意味着计算节点、登录节点和其他集群主机不需要部署这些文件的本地副本。

slurmd在启动时将接触到你指定的slurmctld，配置文件将被拉到该节点。这个slurmctld可以通过一个明确的选项来识别，或者--最好是--通过集群本身定义的DNS SRV记录。

如果你有一个登录节点，你将从该节点运行客户端命令，这些客户端命令将不得不使用DNS记录，以便在运行时从控制器获得配置信息。如果你期望从一个登录节点有大量的流量，这可能会产生大量的配置文件请求。在这种情况下，你可能要考虑在机器上运行slurmd，这样它就可以管理配置文件，但不允许它运行作业。

### 安装

安装这个功能不需要额外的步骤。从Slurm 20.02开始，它是默认内置的。

### 设置

slurmctld必须首先被配置为在无配置模式下运行。这可以通过在`slurm.conf`中设置`SlurmctldParameters=enable_configless`并重新启动slurmctld来处理。

一旦启用，你必须配置slurmd从slurmctld获取其配置。这可以通过使用 `--conf-server` 选项启动 slurmd 来实现，或者通过设置 DNS SRV 记录并确保计算节点上没有本地配置文件。

`--conf-server`选项优先于DNS记录。

命令行选项采用`"$host[:$port]"`，所以一个例子看起来像：
```
slurmd --conf-server slurmctl-primary:6817
```
指定端口是可选的，如果不存在，将默认为6817。多个slurmctlds可以以逗号分隔的列表形式指定，按照优先级顺序（从高到低）。
```
slurmd --conf-server slurmctl-primary:6817,slurmctl-secondary
```
同样的信息可以在DNS的SRV记录中提供。比如说：
```
_slurmctld._tcp 3600 IN SRV 10 0 6817 slurmctl-backup
_slurmctld._tcp 3600 IN SRV 0 0 6817 slurmctl-primary
```
将在启动时向slurmd提供所需的信息。如上所示，如果你在HA设置中部署了Slurm，可以指定多个SRV记录。优先级最低的DNS SRV条目应该是你的主slurmctld，备份slurmctlds的优先级值较高。

### 初始测试

配置好slurmctld并启动slurmd后，你可以在几个地方检查，以确保配置在节点上存在。配置文件将在`/conf-cache/`下的`SlurmdSpoolDir`中，并且在`/run/slurm/conf`中会自动创建一个指向该位置的符号链接。你可以通过在slurmctld节点上的`slurm.conf`中添加一个注释，并运行`scontrol reconfig`，检查配置是否被更新，来确认重载是否有效。

### 限制条件

在 `"SlurmdSpoolDir"`或 `"SlurmdPidFile"`中使用`"%n"`将不能正确替代NodeName，除非slurmd也使用`"-N "`选项启动。

如果使用 systemd 启动 slurmd，必须确保单元文件中没有 `"ConditionPathExists=*"`，否则 slurmd 将无法启动。(Slurm 20.02及以上版本中提供的`slurmd.service`文件的例子不包括这个条目)。

如果任何支持的配置文件 `"Include"`了其他的配置文件，如果 `"Include"`的文件名参考没有路径分隔符，并且该文件位于`slurm.conf`的旁边，那么 `"Include"`的配置文件将只被运出。任何额外的配置文件将需要以不同的方式共享，或添加到父级配置中。

### 注意事项

决定使用何种配置源的优先顺序如下：

1. `slurmd --conf-server $host[:$port]` 选项
1. `-f $config_file` 选项
1. `SLURM_CONF`环境变量（如果设置了）。
1. 默认的slurm配置文件（可能是`/etc/slurm.conf`）。
1. 任何DNS SRV记录（从最低的优先级值到最高）。

支持的配置文件是：

- slurm.conf
- acct_gather.conf
- cgroup.conf
- cli_filter.lua
- ext_sensors.conf
- gres.conf
- helpers.conf
- job_container.conf
- knl_cray.conf
- knl_generic.conf
- mpi.conf
- oci.conf
- plugstack.conf
- topology.conf