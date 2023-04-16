---
title: slurm--selinux
description: slurm中文翻译系列，机翻后纠正了一点，发现其他错误望指出，来源：https://github.com/SchedMD/slurm/blob/master/doc/html/selinux.shtml
published: true
date: 2023-04-16T14:13:12.672Z
tags: slurm
editor: markdown
dateCreated: 2023-04-16T14:13:12.672Z
---

从21.08版本开始，Slurm包括支持为作业设置SELinux上下文，作为技术预览。在未来的版本中，实现方式可能会发生变化，而且默认情况下不支持该功能。

## 架构

当启用时，Slurm作业提交命令--`salloc`、`sbatch`和`srun`--将自动设置一个带有当前操作环境的字段。这个字段可以被`--context`命令行选项所覆盖。

需要注意的是，这个值可以由终端用户直接操作，由特定站点的脚本来验证和控制对这些语境的访问。目前，`MUNGE`（Slurm用户用于安全识别集群上的用户和主机）没有提供一个SELinux上下文字段，因此没有安全的机制来发送当前的上下文到Slurm控制器。因此，在作业提交时提供的上下文，必须由运行在slurmctld中的`job_submit`插件来验证。

如果没有这样的脚本，就不会为用户的作业设置或管理上下文。

## 安装

### 源码：

SELinux支持默认是禁用的，必须在配置时启用。它需要`libselinux1`库和`development headers`文件来构建。
```
configure --enable-selinux
```

## 设置
一旦安装了支持SELinux的Slurm版本，你将需要启用并创建一个`job_submit`插件，在将SELinux上下文传递给slurmctld之前，对其进行验证。目前，还没有一个可靠和安全的方法来获取/验证内部的上下文，所以你必须创建这个脚本，并在`job_submit`插件中执行验证。
例子：
```
function slurm_job_submit(job_desc, part_list, submit_uid)
  if job_desc.req_context then
    local element = 0
    for str in string.gmatch(job_desc.req_context, "([^:]+)") do
      if element == 0 and str ~= "unconfined_u" then
        slurm.log_user("Error: invalid SELinux context")
        return slurm.ERROR
      elseif element == 1 and str ~= "unconfined_r" then
        slurm.log_user("Error: %s is not a valid SELinux role")
        return slurm.ERROR
      end
      element = element + 1
    end
    job_desc.selinux_context = job_desc.req_context
  else
    -- Force a specific context if one wasn't requested
    job_desc.selinux_context = unconfined_u:unconfined_r:slurm_t:s0
  end
  return slurm.SUCCESS
end
```
注意，`job_desc.selinux_context`是根据`job_desc.req_context`的内容设置的，如果它们被认为是有效的。`job_desc.selinux_context`是设置将被使用的上下文。

## 初始测试

`id`对于显示用户当前所处的环境非常有用。作为确保我们正在切换上下文的测试，你可以用`srun`运行一个快速测试。
```
mcmult@master:~$ srun id
uid=1000(mcmult) gid=1000(mcmult) groups=1000(mcmult),27(sudo) context=unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023
mcmult@master:~$ srun --context=unconfined_u:unconfined_r:unconfined_t:s0 id
uid=1000(mcmult) gid=1000(mcmult) groups=1000(mcmult),27(sudo) context=unconfined_u:unconfined_r:unconfined_t:s0
```

## Accounting

目前在Slurm的Accounting中不支持跟踪SELinux上下文。这可能会随着未来版本中支持的发展而改变。如果你需要跟踪SELinux上下文，可以把它存储在管理员的评论栏里，作为`job_submit`插件的一部分，如下述所示。

例子：
```
function slurm_job_submit(job_desc, part_list, submit_uid)
  if job_desc.req_context then
    local element = 0
    for str in string.gmatch(job_desc.req_context, "([^:]+)") do
      if element == 0 and str ~= "unconfined_u" then
        slurm.log_user("Error: invalid SELinux context")
        return slurm.ERROR
      elseif element == 1 and str ~= "unconfined_r" then
        slurm.log_user("Error: %s is not a valid SELinux role")
        return slurm.ERROR
      end
      element = element + 1
    end
    job_desc.selinux_context = job_desc.req_context
  else
    -- Force a specifc context if one wasn't requested
    job_desc.selinux_context = unconfined_u:unconfined_r:slurm_t:s0
  end
  job_desc.admin_comment = "SELinuxContext=" + job_desc.selinux_context
  return slurm.SUCCESS
end
```
注意在返回之前增加了设置 `"job_desc.admin_comment"`。这将设置管理员评论，以显示我们将尝试为该工作设置什么背景。

注意

如果你希望将`pam_slurm_adopt`与SELinux一起使用，请参阅`pam_slurm_adopt`文档，了解如何使之工作。请注意，当同时使用这个功能和`pam_slurm_adopt`时，ssh会话可能不会在与作业相同的上下文中登陆。