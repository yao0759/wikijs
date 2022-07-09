---
title: systemd调试
description: 
published: true
date: 2022-07-09T08:12:55.203Z
tags: systemd
editor: markdown
dateCreated: 2022-06-23T09:30:12.828Z
---

# systemd调试

## 诊断开机问题

如果你的机器在启动过程中卡住了，首先要检查挂起是发生在控制权传递给 systemd 之前还是之后。

尝试在没有 rhgb 和 quiet 的情况下启动内核命令行。如果你看到一些类似这样的信息。

-   Welcome to Fedora _VERSION_ (_codename_)!"  
    
-   Starting _name_...  
    
-   \[ OK \] Started _name_.  
    

则说明 systemd 正在运行。

如果你能得到一个 shell，调试总是变得更容易。如果没有得到登录提示，可以尝试用CTRL+ALT+F\_\_切换到其他虚拟终端。显示服务器启动的问题可能表现为tty1上没有登录，但其他VT可以工作。

如果启动时没有在任何一个虚拟控制台上显示登录信息就停止了，在宣布它肯定卡住之前，让它重试最多5分钟。有一种可能是启动困难的服务在这个超时后会被杀死，启动会继续正常进行。另一种可能性是，一个重要的挂载点的设备将无法出现，你将会看到紧急模式。

### 假如没有shell

如果你既没有得到正常的登录，也没有得到紧急模式的外壳，你将需要做额外的步骤来从机器中获得调试信息。

-   尝试CTRL+ALT+DEL重启

-   用SysRq或硬重置强制重启。  
    

-   当下次启动时，你将不得不添加一些内核命令行参数，这取决于你从下面的选项中选择哪种调试策略。  
    

#### 调试记录到串行控制台

如果你有一个硬件串口控制台，或者你在虚拟机中进行调试（例如，使用virt-manager，你可以在菜单View -> Text Consoles中切换到串口控制台，或者使用virsh console MACHINE从终端连接），你可以要求systemd在启动时记录大量有用的调试信息。

```
systemd.log_level=debug systemd.log_target=console console=ttyS0,38400 console=tty1
```

如果pid 1出现故障，上述方法很有用，但如果稍后但关键的启动服务出现故障（如网络），可以通过以下方法配置journald转发到控制台。

systemd.journald.forward\_to\_console=1 console=ttyS0,38400 console=tty1

console=可以指定多次，systemd会输出到所有的控制台。

#### 启动到救援目标或紧急目标

在内核命令行中添加 systemd.unit=rescue.target 或只添加 1 来直接启动救援目标。如果问题发生在基本系统启动后，在启动 "正常 "服务的过程中，这个目标就很有用。如果是这种情况，你应该能够从这里禁用坏的服务。如果救援目标也不能启动，更小的应急目标可能会启动。

在内核命令行中添加 systemd.unit=emergency.target 或 emergency，可以直接启动到 emergency shell。请注意，在紧急情况下，在编辑任何文件之前，你必须自己重新挂载根文件系统的读写器。

在紧急状态下可以解决的常见问题是/etc/fstab中的问题挂载项。修复 /etc/fstab 后，运行 systemctl daemon-reload，让 systemd 刷新它的视图。

如果连应急目标都不能工作，你可以直接用 init=/bin/sh 启动到 shell。如果 systemd 本身或其依赖的某些库被文件系统损坏，这可能是必要的。你可能需要重新安装受影响软件包的工作版本。

如果 init=/bin/sh 不起作用，你必须从其他介质启动。

### 尽快打开调试shell

你可以在启动过程中尽早启用shell权限，以便利用各种systemctl命令诊断systemd相关的启动问题。使用以下命令启用它

```
systemctl enable debug-shell.service
```

或指定

或在内核命令行中指定 systemd.debug-shell=1。

小贴士：如果你发现自己无法使用 systemctl 与运行中的 systemd 进行通信（例如从不同的启动系统中设置），你可以通过指定 --root= 来避免与管理器通信。

```
systemctl --root=/ enable debug-shell.service
```

一旦启用，下次启动时就可以用CTRL+ALT+F9切换到tty9，在启动过程的早期就有一个root shell可用。你可以用这个shell来检查服务的状态，阅读日志，用systemctl list-jobs查找卡住的工作，等等。

警告。警告：这个shell只能用于调试！不要忘记关闭systemd的功能。在调试完开机问题后，不要忘记禁用 systemd-debug-shell.service。让root shell一直可用会有安全隐患。

也可以将kbrequest.target别名为debug-shell.service，以便按需启动调试外壳。这有同样的安全问题，但可以避免一直运行shell。

#### 验证先决条件

需要有一个（至少是部分）填充的/dev。根据你的设置（例如在嵌入式系统上），检查Linux内核配置选项CONFIG\_DEVTMPFS和CONFIG\_DEVTMPFS\_MOUNT是否被设置。另外，为了使操作无误，建议支持cgroups和fanotify，所以检查Linux内核配置选项CONFIG\_CGROUPS和CONFIG\_FANOTIFY是否被设置。消息 "Failed to get D-Bus connection: 在各种systemctl操作中，出现 "Failed to get D-Bus connection: No connection to service manager. "的提示，说明缺少这些选项。

## 假如有shell

当 systemd 运行到可以为你提供 shell 的程度时，请用它来提取有用的信息进行调试。在内核命令行上用这些参数启动。

**systemd.log\_level=debug systemd.log\_target=kmsg log\_buf\_len=1M printk.devkmsg=on**以提高 systemd 的粗暴程度，让 systemd 把日志写到内核日志缓冲区，增加内核日志缓冲区的大小，并防止内核丢弃信息。到达 shell 后，看一下日志。

当报告一个bug时，用管道将其传送到一个文件，并将其附在bug报告中。

要检查可能被卡住的作业，请使用。

被列为 "正在运行 "的作业是在 "等待 "的作业被允许开始执行之前必须完成的。

## 诊断关机问题

就像开机问题一样，当你在关机过程中遇到挂起时，确保你至少等待5分钟，以区分永久性的挂起和只是超时的坏服务。然后值得测试的是，系统是否对CTRL+ALT+DEL有任何反应。

如果你的系统关机（无论是重启还是断电）被卡住了，首先测试内核本身是否能够使用这些命令来强制重启或断电。

如果这两个命令中的任何一个都不起作用，那就很可能是内核的问题，而不是 systemd 的问题。

## 关机最终完成

如果正常的重启或关机工作，但花费的时间可疑地长，那么

-   用调试选项启动。  
    

```
systemd.log_level=debug systemd.log_target=kmsg log_buf_len=1M printk.devkmsg=on enforcing=0
```

-   将以下脚本保存为/usr/lib/systemd/system-shutdown/debug.sh并使其可执行。  
    

```

mount -o remount,rw /
dmesg > /shutdown-log.txt
mount -o remount,ro /
```

-   重新启动  
    

寻找记录在结果文件**shutdown-log.txt**中的超时情况，并/或将其附在bugreport中。

## 关机从未完成

如果正常的重启或关机即使在等待几分钟后也从未完成，那么上述创建关机日志的方法将无济于事，必须使用其他方法获得日志。有两个对调试启动问题有用的选项也可以用于关机问题。

-   使用串行控制台  
    
-   使用debug shell--它不仅从早期启动时就可以使用，而且一直到晚期关机时都处于活动状态。  
    

### 服务的状态和日志

当服务启动失败时，systemctl会给你一个通用的错误信息：

```

Job failed. See system journal and 'systemctl status' for details.
```

该服务可能已经打印了自己的错误信息，但你没有看到，因为由 systemd 运行的服务与你的登录会话无关，它们的输出没有连接到你的终端。但这并不意味着输出丢失。默认情况下，服务的 stdout 和 stderr 都指向 systemd 日志，服务通过 syslog(3) 生成的日志也会进入该日志，systemd 还会保存失败服务的退出代码。我们来看看。

```

foo.service - mmm service
          Loaded: loaded (/etc/systemd/system/foo.service; static)
          Active: failed (Result: exit-code) since Fri, 11 May 2012 20:26:23 +0200; 4s ago
         Process: 1329 ExecStart=/usr/local/bin/foo (code=exited, status=1/FAILURE)
          CGroup: name=systemd:/system/foo.service
May 11 20:26:23 scratch foo[1329]: Failed to parse config
```

在这个例子中，该服务以PID为1329的进程运行，并以错误代码1退出。如果你以 root 或 adm 组的用户身份运行 systemctl status，你会从该服务写的日志中得到几行。在这个例子中，该服务只产生了一条错误信息。

要列出日志，请使用 journalctl 命令。

如果你有一个syslog服务（比如rsyslog）在运行，日志也会将信息转发给它，所以你会在/var/log/messages中找到它们（取决于rsyslog的配置）。