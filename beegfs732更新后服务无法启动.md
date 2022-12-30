---
title: beegfs 7.3.2更新后服务无法启动
description: 
published: true
date: 2022-12-30T15:07:46.624Z
tags: beegfs
editor: markdown
dateCreated: 2022-12-30T15:07:46.624Z
---



beegfs 7.3.2版本默认强制身份验证身份。所以在安装或升级后，没有配置authfile会导致服务无法启动。

解决办法有两种：

1. 禁用`connDisableAuthentication`(不推荐)

```
vim /etc/beegfs/beegfs-mgmtd.conf
......
connDisableAuthentication              = true
......
```

1. 创建共享密钥

使用dd创建一个随机密钥

```
dd if=/dev/random of=/etc/beegfs/connauthfile bs=128 count=1
```

修改文件所属者为root并只读

```
chown root:root /etc/beegfs/connauthfile
chmod 400 /etc/beegfs/connauthfile
```

然后修改各个服务的配置文件，一般有：

- beegfs-client.conf
- beegfs-helperd.conf
- beegfs-meta.conf
- beegfs-mgmtd.conf
- beegfs-storage.conf

修改上述配置文件中的

```
vim /etc/beegfs/beegfs-***.conf
......
connAuthFile                           = /etc/beegfs/connauthfile
......
```

重启服务

```
systemctl restart beegfs-mgmtd.service
systemctl restart beegfs-meta.service
systemctl restart beegfs-storage.service
systemctl restart beegfs-client.service
```