---
title: beegfs客户端编译以支持RDMA
description: 
published: true
date: 2022-12-30T14:54:18.225Z
tags: beegfs, rdma
editor: markdown
dateCreated: 2022-12-30T14:54:18.225Z
---


beegfs客户端在不同发行版上支持的OFED版本是不同的，像我在ubuntu2004上发现beegfs对我手上的MCX353A-FCBT并不支持，因为MCX353A-FCBT是相对比较旧的网卡，从从MLNX_OFED 5.1开始，就不再继续支持了。要查询发行版本对你的网卡的支持，可以查看下述[信息](https://doc.beegfs.io/latest/release_notes.html?highlight=ofed)：

> - RHEL 8.3: no OFED, OFED 4.9, 5.0, 5.1, 5.2, 5.3, 5.4
> - AlmaLinux 8.4: no OFED, OFED 5.3, 5.4
> - AlmaLinux 8.5: no OFED, OFED 5.3, 5.4, 5.5
> - AlmaLinux 9.0: no OFED, OFED 5.6, 5.7
> - Rocky Linux 8.4: no OFED, OFED 5.3, 5.4
> - Rocky Linux 8.5: no OFED, OFED 5.5
> - Rocky Linux 8.6: no OFED, OFED 5.6
> - SLES 15.1: no OFED, OFED 5.0
> - SLES 15.2: no OFED, OFED 5.1, 5.4
> - SLES 15.3: no OFED, OFED 5.4, 5.5, 5.6
> - SLES 15.4: OFED 5.6, 5.7
> - Debian 10: no OFED, OFED 5.2, 5.3, 5.4
> - Debian 11: no OFED, OFED 5.6
> - Ubuntu 18.04: no OFED
> - Ubuntu 20.04: no OFED, OFED 5.4
> - Ubuntu 22.04: no OFED, OFED 5.6, 5.7

因此，假如需要RDMA的支持，我们还需要重新编译一下客户端，在此之前，需要先安装官方驱动，具体方法可以参考下述[链接](https://developer.aliyun.com/article/1003056?spm=a2c6h.26396819.creator-center.20.73863e1875X8nA)。

安装完成后，修改/etc/beegfs/beegfs-client-autobuild.conf，找到buildArgs=-j8这一行，修改为下述信息

```
buildArgs=-j8 BEEGFS_OPENTK_IBVERBS=1 OFED_INCLUDE_PATH=/usr/src/ofa_kernel/default/include/
```

然后执行

```
/etc/init.d/beegfs-client rebuild
```

然后需要配置mgmtd和metadata和storage。检查一下，假如storage显示是RDMA，client应该也没有太大问题了，可以放心。

```
beegfs-net
```