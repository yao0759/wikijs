---
title: beegfs定期备份元数据和mgmt
description: 
published: true
date: 2022-12-30T14:59:20.877Z
tags: beegfs
editor: markdown
dateCreated: 2022-12-30T14:59:20.877Z
---

## 为什么不使用buddy group还需要备份元数据

首先官方文档中，提到一个问题，就是buddy group只适用于磁盘故障、服务器故障、网络故障等问题，但是当系统降级它并不能提供很好的保护，如果有buddy group处于降级状态，则可能会导致数据丢失。而且文件被用户或进程意外删除或覆盖，buddy group不会帮助你把旧文件找回来。所以你仍然有责任对重要的目录进行定期备份。

## 使用borgbackup备份元数据

beegfs推荐使用borgbackup增量备份工具。

先初始化仓库

```
root@storage01:~# borg init --encryption=keyfile /root/backup/
Enter new passphrase:
Enter same passphrase again:
Do you want your passphrase to be displayed for verification? [yN]: y
Your passphrase (between double-quotes): "***********"
Make sure the passphrase displayed above is exactly what you wanted.

By default repositories initialized with this version will produce security
errors if written to with an older version (up to and including Borg 1.0.8).

If you want to use these older versions, you can disable the check by running:
borg upgrade --disable-tam /root/backup

See https://borgbackup.readthedocs.io/en/stable/changes.html#pre-1-0-9-manifest-spoofing-vulnerability for details about the security implications.

IMPORTANT: you will need both KEY AND PASSPHRASE to access this repo!
Use "borg key export" to export the key, optionally in printable format.
Write down the passphrase. Store both at safe place(s).
```

备份beegfs metadata

```
root@storage01:~# borg create --stats --progress /root/backup::22-12-2022 /beegfs_metadata/
Enter passphrase for key /root/.config/borg/keys/root_backup:
------------------------------------------------------------------------------
Archive name: 22-12-2022
Archive fingerprint: 2701d8f59c6a21d161d55745a0aae3872105de6bceb3772369f46651433368ef
Time (start): Wed, 2022-12-21 14:57:28
Time (end):   Wed, 2022-12-21 14:57:35
Duration: 7.51 seconds
Number of files: 29
Utilization of max. archive size: 0%
------------------------------------------------------------------------------
                       Original size      Compressed size    Deduplicated size
This archive:                8.29 MB            664.17 kB            664.12 kB
All archives:                8.29 MB            664.97 kB            664.93 kB

                       Unique chunks         Total chunks
Chunk index:                      57                   58
------------------------------------------------------------------------------
```

## mgmtd备份

mgmtd除了可以用上述的方法备份目录，这里其实有两种方法，首先是mgmtd服务对buddy的迁移有重要的作用，可以将其备份到各个节点上，也可以选择DRDB进行远程备份，这部分在官方文档中也有提到

> - **BeeGFS-mgmtd backup**: We strongly recommend to setup a backup solution like rsync, in order to backup the management target (often /beegfs/mgmtd/) to a safe external location. Please keep in mind that RAID or buddy mirroring is no replacement for an external backup. Also, you can set up your beegfs-mgmtd in a DRDB HA setup as described in https://www.beegfs.io/login/wiki2/MgmtdHAWithDRBD , to have valid 2nd. copy of the BeeGFS management data.