---
title: redhat 9 仓库报错修复
description: 
published: true
date: 2023-01-27T04:28:05.725Z
tags: redhat
editor: markdown
dateCreated: 2023-01-27T04:28:05.725Z
---

我在redhat 9中更新package，出现下述报错
```shell
[root@localhost ~]# dnf update       
Updating Subscription Management repositories.
Error: There are no enabled repositories in "/etc/yum.repos.d", "/etc/yum/repos.d", "/etc/distro.repos.d"
```

根据这个[参考链接](https://access.redhat.com/discussions/6394941) 找到了解决方法。

1. 取消注册
   ```shell
   sudo subscription-manager remove --all
   sudo subscription-manager unregister
   sudo subscription-manager clean
   ```
   
2. 重新注册
   ```shell
   sudo subscription-manager register
   sudo subscription-manager refresh
   ```
   
3. 查找pool id
   ```shell
   sudo subscription-manager list --available
   ```
   
4. 附加订阅
   ```shell
   sudo subscription-manager attach --pool=<Pool-ID>
   ```