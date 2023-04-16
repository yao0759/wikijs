---
title: slurm--用户权限
description: slurm中文翻译系列，机翻后纠正了一点，发现其他错误望指出，来源：https://github.com/SchedMD/slurm/blob/master/doc/html/user_permissions.shtml
published: true
date: 2023-04-16T13:26:59.886Z
tags: slurm
editor: markdown
dateCreated: 2023-04-16T13:26:59.886Z
---

Slurm支持几种特殊的用户权限，如下所述。

## Operator
这些用户可以添加、修改和删除任何数据库对象（用户、账户等），并添加其他操作员。在一个SlurmDBD服务的集群上，这些用户可以

- 查看被`PrivateData`标志阻挡在常规用途之外的信息
- 创建/改变/删除保留
使用用户数据库记录中的`AdminLevel`选项来设置。有关配置信息，请参阅会计和资源限制。

## Admin

这些用户在数据库中具有与操作员相同的权限水平。他们也可以改变服务的slurmctld上的任何东西，就像他们是SlurmUser或root一样。

在用户的数据库记录中有一个`AdminLevel`选项。有关配置信息，请参阅会计和资源限制。

## Coordinator
一个特殊的特权用户，通常是一个账户经理，可以向他们所负责的账户添加用户或子账户。这应该是一个受信任的人，因为他们可以改变账户和用户关联的限制，以及取消、重新申请或重新分配他们领域内的工作账户。

设置使用Slurm的数据库中的表，定义用户和账户，他们可以作为协调人。有关配置信息，请参见 `sacctmgr` man 页。
