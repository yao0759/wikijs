---
title:  0 0 0 ubuntu 20.04如何沙箱化systemd服务
description: 
published: true
date: 2022-12-30T15:29:20.998Z
tags: systemd, ubuntu, sandbox
editor: markdown
dateCreated: 2022-12-30T15:29:20.998Z
---

随着systemd越来越成熟，systemd提供了很多的功能特性可以对进程进行资源隔离和防护，虽然不是完全的隔离性，但是还是为安全性提供了保障。

这里我们使用vncserver systemd服务来做演示，这里的vncserver systemd配置如下

```
[Unit]
Description=Remote desktop service (VNC)
After=syslog.target network.target
[Service]
Type=simple
ExecStartPre=/bin/bash -c '/usr/bin/vncserver -kill ":$(id -u %i)" > /dev/null 2>&1 || :'
ExecStart=/bin/bash -c 'cd /home/%i;/usr/bin/vncserver ":$(id -u %i)" -geometry 1440x900 -alwaysshared -fg'
ExecStop=/bin/bash -c '/usr/bin/vncserver -kill ":$(id -u %i)"'
[Install]
WantedBy=graphical.target
```

将上述的进程沙箱化的话我们可以执行下述步骤：

1. 配置进程执行的用户和用户组

```
[Service]
User=%i
Group=%i
```

2. 禁止服务进程及其子进程获取新的权限

```
NoNewPrivileges=true
```

3. 阻止进程获取内核变量

```
ProtectKernelTunables=true
```

4. 阻止进程加载或者卸载内核模块

```
ProtectKernelModules=true
```

5. 阻止进程读取或者写入内核日志

```
ProtectKernelLogs=true
```

6. 阻止修改cgroup

```
ProtectControlGroups=true
```

7. 阻止进程修改系统内存中的任何代码

```
MemoryDenyWriteExecute=true
```

8. 阻止进程对文件或者目录设置SUID或者SGID

```
RestrictSUIDSGID=true
```

9. 阻止进程修改硬件或者软件的时钟

```
ProtectClock=true
```

禁用进程启用实时调度，防止cpu过载

```
RestrictRealtime=true
```

11. 强制进程使用特定目录。这可以防止进程读取其他程序在临时文件夹的内容

```
PrivateTmp=true
```