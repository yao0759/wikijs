---
title: beegfs高可用模式探讨
description: 
published: true
date: 2022-07-09T08:11:46.253Z
tags: beegfs, 高可用
editor: markdown
dateCreated: 2022-06-23T09:20:59.312Z
---

**简介：** beegfs高可用模式探讨

最近在测试beegfs，它在hpc应用十分广泛的并行文件系统，但是它保证数据安全性的方式只有mirror一种方式，这种方式无法磁盘的有效使用率较低。模式如下图所示

![屏幕截图 2022-01-11 215313.png](https://ucc.alicdn.com/pic/developer-ecology/c40b785e921043109b43170cc361d5d0.png "屏幕截图 2022-01-11 215313.png")

netapp在磁盘共享架构下，实现了单节点掉线后依然能够保证服务的有效访问，这是因为它们的硬盘实际是通过存储阵列柜上共享到节点上，因此其中一个节点掉线后，另一个节点通过存储阵列柜依然能够访问掉线节点的硬盘。但是并不是仅仅能够访问硬盘就能添加到beegfs服务中去，还需要做些操作。我进行了一下测试，当我把掉线节点的硬盘添加到在线节点上时，还需要对以下文件做出修改才行。

首先需要挂载掉线节点硬盘

```
mount /dev/lose-disk1 /mnt/data*
```

挂载完成后，需要修改/etc/beegfs/beegfs-storage.conf配置文件，主要修改

```
storeStorageDirectory和storeFsUUID，将掉线节点的挂载目录和UUID添加
storeStorageDirectory        = ,/mnt/data1 ,/mnt/data2 ,/mnt/data3 ,/mnt/data4
storeFsUUID                  = ,1c80fdd1-4e81-4463-80fb-bd948265e98d  ,050c19f1-0fd9-4725-b21c-88fe13a37f5c ,1e3392ce-b274-4717-979e-b4ac183c812f ,d4f87f8b-b7a3-4297-9824-4fef30e90920
```

然后还需要到挂载目录下，修改nodeNumID和originalNodeID

nodeNumID要修改还存活的节点的ID，如node01是1，node02是2，node02掉线，node02上的硬盘就要修改为1；originalNodeID则要将掉线节点修改为存活节点，如node02修改为node01

做完上述操作，尝试**systemctl restart beegfs-storage**，即可恢复。

因此我们可以写一个脚本，自动化上述操作，也能实现类似的效果

