---
title: slurm--pam_slurm_adopt
description: 
published: true
date: 2023-07-17T14:42:19.080Z
tags: slurm
editor: markdown
dateCreated: 2023-07-17T14:42:19.080Z
---

# pam_slurm_adopt

该模块的目的是防止用户ssh进入没有运行任务的节点，并跟踪ssh连接和任何其他产生的进程，以进行会计核算，并确保在任务完成后彻底清理任务。该模块通过确定发起ssh连接的作业来实现这一功能。用户的连接被`adopted`到作业的`external`步骤中。

## 安装

### 源码

在您的 Slurm 源代码目录下，导航到` ./contribs/pam_slurm_adopt/` 并运行

```
make && make install
```

作为 root。这将把 `pam_slurm_adopt.a`、`pam_slurm_adopt.la` 和 `pam_slurm_adopt.so` 放在 `/lib/security/` (Debian 系统) 或 `/lib64/security/` (RedHat/SuSE 系统)。

### RPM

包含的 `slurm.spec` 将生成 slurm-pam_slurm RPM，该 RPM 将安装 `pam_slurm_adopt`。有关管理基于 RPM 的安装的说明，请参阅 [Quick Start Administrator Guide](https://slurm.schedmd.com/quickstart_admin.html)。

## PAM配置

将下面一行添加到`/etc/pam.d`中的相应文件，如system-auth或sshd（可以使用 `required`或 `sufficient`PAM控制标志）：

```
account    required      pam_slurm_adopt.so
```

插件的顺序非常重要，`pam_slurm_adopt.so`应该是账户堆栈中最后一个PAM模块。包含的文件如common-account通常应该在pam_slurm_adopt之前。在 sshd 中可能有如下的帐户堆栈：

```
account    required      pam_nologin.so
account    include       password-auth
...
-account    required      pam_slurm_adopt.so
```

注意在pam_slurm_adopt的账户条目前的`"-"`。如果找不到 `pam_slurm_adopt.so` 文件，它允许 PAM 从容失败。如果Slurm在共享文件系统上，例如NFS，那么建议这样做，以避免在共享文件系统挂载或宕机时被锁定在节点之外。

pam_slurm_adopt 必须与 `task/cgroup` 任务插件和 `proctrack/cgroup` 或 `proctrack/cray_aries` proctrack 插件一起使用。pam_systemd模块会与pam_slurm_adopt冲突，因此需要在所有包含在sshd或system-auth中的文件（如password-auth、common-session等）中禁用它。

如果需要pam_systemd的用户管理功能，例如处理用户运行时目录`/run/user/$UID`，可以让prolog脚本运行`loginctl enable-linger $SLURM_JOB_USER`，然后在epilog脚本中运行`loginctl disable-linger $SLURM_JOB_USER`，再次禁用它（在确保节点上没有来自该用户的其他作业之后）。如果您的软件需要` XDG_* `环境变量，您还需要导出这些变量。你可以在这里看到prolog和epilog脚本的例子：

Prolog：

```
loginctl enable-linger $SLURM_JOB_USER
exit 0
```

TaskProlog:

```
echo "export XDG_RUNTIME_DIR=/run/user/$SLURM_JOB_UID"
echo "export XDG_SESSION_ID=$(</proc/self/sessionid)"
echo "export XDG_SESSION_TYPE=tty"
echo "export XDG_SESSION_CLASS=user"
```

Epilog:

```
#Only disable linger if this is the last job running for this user.
O_P=0
for pid in $(scontrol listpids | awk -v jid=$SLURM_JOB_ID 'NR!=1 { if ($2 != jid && $1 != "-1"){print $1} }'); do
        ps --noheader -o euser p $pid | grep -q $SLURM_JOB_USER || O_P=1
done
if [ $O_P -eq 0 ]; then
        loginctl disable-linger $SLURM_JOB_USER
fi
exit 0
```

你还必须确保不同的PAM模块在进入`pam_slurm_adopt.so`之前没有短路。在上面的例子中，下面两行在password-auth文件中被注释掉了：

```
#account    sufficient    pam_localuser.so
#-session   optional      pam_systemd.so
```

**注意：这可能需要编辑一个自动生成的文件。不要运行生成该文件的配置脚本，否则你的修改将被删除。**

如果你总是希望允许一个管理组（例如wheel）访问，在pam_slurm_adopt之后堆叠pam_access模块。pam_slurm_adopt 成功就足以允许访问，但是 pam_access 模块可以允许其他人，例如管理员，即使没有在该节点上的工作也可以访问：

```
account    sufficient    pam_slurm_adopt.so
account    required      pam_access.so
```

然后编辑pam_access配置文件（`/etc/security/access.conf`）：

```
+:wheel:ALL
-:ALL:ALL
```

除了 pam_access 之外，还可以将 `pam_listfile.so` 放在 `pam_slurm_adopt.so` 之前。例如

```
account    sufficient    pam_listfile.so item=user sense=allow onerr=fail file=/path/to/allowed_users_file
account    required      pam_slurm_adopt.so
```

在allowed_users_file中列出允许访问的用户的用户名。

当访问被拒绝时，用户将收到相关的错误信息。

## pam_slurm_adopt Module Options 

该模块是可配置的。在`/etc/pam.d/`（例如sshd或system-auth）中相应文件的pam_slurm_adopt行末尾添加这些选项：

```
account sufficient pam_slurm_adopt.so optionname=optionvalue
```

This module has the following options:

- **action_no_jobs**

  如果用户在节点上没有作业，则执行的操作。可配置的值：

  - **ignore** 什么都不做；

  - **deny**（默认）拒绝连接。

- **action_unknown**

  当用户在节点上有多个作业，而*RPC没有找到源作业时执行的操作。如果 RPC 机制在您的环境中正常工作，则此选项可能仅在从登录节点连接时*相关。可配置的值：

  **newest**（默认值）在使用*cgroup/v1*的系统上，选择节点上最新的作业。选择 "newest"作业的依据是作业的 step_extern cgroup 的 mtime；询问 Slurm 需要向控制器发送 RPC。因此，内存 cgroup 必须在使用中，以便代码可以检查 cgroup 目录的 mtime。用户可以 ssh 登录，但可能会进入一个比他们要检查的作业更早退出的作业。ssh连接至少会受到适当的限制，如果这成为一个问题，用户可以被告知更好的方法来实现他们的目标。

  **注意**： 如果模块无法获取cgroup mtime，那么选中的作业可能不是最新的。在使用*cgroup/v2*的系统中，最新的工作只是id最大的工作，因此这并不能确保它确实是最新的工作。

- **action_adopt_failure**

  如果进程由于某种原因无法被任何作业采用，将执行的操作。如果进程不能被callerid RPC识别的作业所采用，它将跳转到action_unknown代码并尝试在那里采用。如果在这一点上失败或只有一个作业，将导致执行此操作。可配置的值：

  - **allow** (默认)允许连接通过而不采用。**警告：此值不安全，建议仅用于测试目的。我们建议使用 "deny. "**
  - **deny**拒绝连接。

- **action_generic_failure**

  如果出现某些故障，例如无法与本地*slurmd*对话，或者内核没有提供正确的设施，将执行的操作。可配置的值：

  - **ignore**（默认）不执行任何操作。跳转到下一个pam模块。**警告：此值不安全，建议仅用于测试目的。我们建议使用 "deny. "**

  - **allow**允许连接通过而不通过。

  - **deny**拒绝连接。

- **disable_x11**

  关闭 Slurm 内置的 X11 转发支持。可配置的值：

  - **0** (默认)如果连接被采用的作业启用了Slurm的X11转发，DISPLAY变量将被X11隧道端点细节覆盖。

  - **1**不检查Slurm的X11转发支持，也不改变DISPLAY变量。

- **join_container**

  控制与 `job_container/tmpfs` 插件的交互。可配置的值：

  - **true**（默认）尝试加入 `job_container/tmpfs` 插件创建的容器。

  - **false** 不尝试加入容器。

- **log_level**

  有关可用选项，请参见 slurm.conf 中的 [SlurmdDebug](https://slurm.schedmd.com/slurm.conf.html#OPT_SlurmdDebug)。默认的log_level是**info**。

- **nodename**

  如果 **slurm.conf** 中定义的 NodeName 与此节点的主机名不同（如 **hostname -s** 所报告的），则必须将其设置为 **slurm.conf** 中此主机作为的 NodeName。

- **service**

  该模块应该运行的 pam 服务名称。默认情况下，它只运行在sshd上。可以指定不同的服务名，如`login`或`*`，以允许模块在任何服务上下文中运行。对于本地pam登录，该模块可能会导致意外行为甚至安全问题。因此，如果服务名称不匹配，该模块将不执行通过逻辑，并立即返回`PAM_IGNORE`。

## Slurm配置

必须在 `slurm.conf` 中设置 **PrologFlags=contain**。这将设置 "external"步骤，其中将采用 ssh 启动的进程。您还必须在 `slurm.conf` 中启用 `task/cgroup` 插件。参见 [Slurm cgroups 指南](https://slurm.schedmd.com/cgroups.html)

### **IMPORTANT** 

在使用本模块之前，**PrologFlags=contain**必须就位。模块基于已经启动的本地步骤进行检查。如果用户在节点上没有任何步骤，例如外部步骤，模块将假定用户没有分配给节点的作业。根据您对PAM模块的配置，您可能会意外地拒绝所有没有**PrologFlags=contain.**的用户的ssh尝试。

slurm.conf中的**UsePAM**选项与pam_slurm_adopt无关。

## 其他配置

确认`/etc/ssh/sshd_config`中的**UsePAM**是否设置为**On**（默认情况下应该为打开）。

通过设置**LaunchParameters=ulimit_pam_adopt**，外部步骤采用的进程可以设置`RLIMIT_RSS`，类似于在常规步骤中运行的任务。

### FIREWALLS, IP ADDRESSES, ETC. 

*slurmd* 应该可以在任何用户可以启动 ssh 的 IP 地址上访问。确定源作业的 RPC 必须能够到达该特定 IP 地址上的 *slurmd* 端口。如果源节点上没有 *slurmd*，例如在登录节点上，最好是拒绝 RPC 而不是静默丢弃。这样可以更好地响应 RPC 发起者。

### SELINUX

SELinux 可能会与 pam_slurm_adopt 冲突，但一般情况下它们可以并肩工作。这是一个在 Debian 系统中使用的类型强制文件示例。它提供了一些指导，并展示了实现此功能所需的条件，但可能需要额外的修改。

```
module pam_slurm_adopt 1.0;

require {
	type sshd_t;
	type var_spool_t;
	type unconfined_t;
	type initrc_var_run_t;
	class sock_file write;
	class dir { read search };
	class unix_stream_socket connectto;
}

#============= sshd_t ==============
allow sshd_t initrc_var_run_t:dir search;
allow sshd_t initrc_var_run_t:sock_file write;
allow sshd_t unconfined_t:unix_stream_socket connectto;
allow sshd_t var_spool_t:dir read;
allow sshd_t var_spool_t:sock_file write;
```

有些插件可能需要比这更多的权限。值得注意的是，`job_container/tmpfs`需要类似这样的权限：

```
module pam_slurm_adopt 1.0;

require {
	type nsfs_t;
	type var_spool_t;
	type initrc_var_run_t;
	type unconfined_t;
	type sshd_t;
	class sock_file write;
	class dir { read search };
	class unix_stream_socket connectto;
	class fd use;
	class file read;
	class capability sys_admin;
}

#============= sshd_t ==============
allow sshd_t initrc_var_run_t:dir search;
allow sshd_t initrc_var_run_t:sock_file write;
allow sshd_t nsfs_t:file read;
allow sshd_t unconfined_t:fd use;
allow sshd_t unconfined_t:unix_stream_socket connectto;
allow sshd_t var_spool_t:dir read;
allow sshd_t var_spool_t:sock_file write;
allow sshd_t self:capability sys_admin;
```

## 限制条件

使用 pam_slurm_adopt 可能会破坏进程采用多因素身份验证等其他身份验证方法。

当在 Slurm 中使用 SELinux 支持时，通过 pam_slurm_adopt 启动的会话不一定与它关联的作业处于相同的上下文中。