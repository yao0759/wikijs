---
title: slurm--工作量特性化密钥（WCKey）
description: slurm中文翻译系列，机翻后纠正了一点，发现其他错误望指出，来源：https://github.com/SchedMD/slurm/blob/master/doc/html/wckey.shtml
published: true
date: 2023-04-16T13:22:44.539Z
tags: slurm
editor: markdown
dateCreated: 2023-04-16T13:18:30.593Z
---

# slurm--工作量特性化密钥（WCKey）管理

WCKey是一种正交的方式，用于对可能不相关的账户进行核算。这在来自不同账户的用户都在同一个项目上工作的情况下很有用。

## slurm(dbd).conf设置

在slurm.conf文件的`AccountingStorageEnforce`选项中包括 `"WCKey"`将强制执行每个作业的WCKeys。这意味着只有具有有效WCKeys（之前通过`sacctmgr`添加的WCKeys）的作业才会被允许运行。

如果你想跟踪作业的WCKey值，你必须在`slurm.conf`和`slurmdbd.conf`文件中设置`TrackWCKey`选项。这将确保WCKey在每个作业中被跟踪。如果你在`AccountingStorageEnforce`行中设置了 `"WCKey"`，TrackWCKey就会被自动设置，但它仍然需要被添加到`slurmdbd.conf`文件中。

## sbatch/salloc/srun

每个提交工具都有`--wckey=`选项，可以为一个作业设置WCKey。`[SBATCH|SALLOC|SLURM]_WCKEY`也可以在环境中设置WCKey。如果没有给出WCKey，作业的WCKey将被设置为用户对集群的默认WCKey，这可以通过`sacctmgr`设置。同样，如果没有指定WCKey，会计记录会附加一个 "*"，表示没有指定WCKey。这对管理员确定一个用户是否指定了他们的WCKey很有用。

## Sacct

Sacct可以通过在`--format`选项中添加 `"wckey"`来查看WCKey。你也可以通过使用`--wckeys=`选项来单独列出作业，这将只发送以特定WCKeys运行的作业信息。

## sacctmgr

Sacctmgr 用来管理WCKeys。你可以添加和删除用户的WCKeys或者列出它们。

你将一个用户添加到一个WCKey中，就像你添加一个帐户一样，只是WCKey不需要事先创建。

```
sacctmgr add user da wckey=secret_project
```

你可以用同样的方式从WCKey中删除它们。

```
sacctmgr del user da wckey=secret_project
```

要改变用户的默认WCKey，你可以运行以下一行

```
sacctmgr mod user da cluster=snowflake set defaultwckey=secret_project
```

这将改变集群 "snowflake "上的用户 "da "的默认WCKey为 `"secret_project"`。如果你想让所有集群都这样，只需删除`cluster=`选项。

## sreport

关于WCKeys可用的报告的信息可以在sreport手册中找到。