---
title: ubuntu2004安装corosync和pacemaker并为集群添加浮动IP
description: 
published: true
date: 2022-12-30T14:48:37.624Z
tags: 高可用, ubuntu, pacemaker, corosync
editor: markdown
dateCreated: 2022-12-30T14:48:37.624Z
---

环境

| 节点      | IP            |
| --------- | ------------- |
| storage01 | 10.141.161.11 |
| storage02 | 10.141.161.12 |

安装包（两个节点都要安装）

```
apt install pacemaker corosync pcs
```

启用pcsd服务（两个节点都要执行）

```
systemctl enable --now pcsd
```

配置hacluster密码（两个节点都要执行）

```
root@storage01:~# passwd hacluster
Changing password for user hacluster.
New password:
Retype new password:
passwd: all authentication tokens updated successfully.
```

组成cluster，仅在其中一个节点上执行即可，因为我是首次添加，增加–force选项，可以避免不少问题

```
pcs cluster setup cluster storage01 storage02 --force
```

开启服务，仅在其中一个节点上执行即可

```
pcs cluster start --all
```

启用服务，仅在其中一个节点上执行即可

```
pcs cluster enable --all
```

查看集群状态，假如跟下述一样，显示有某个节点offline，可以选择等待一下，我这两个节点花了一点时间才同步的

```
root@storage01:~# pcs cluster status
Cluster Status:
 Cluster Summary:
   * Stack: corosync
   * Current DC: storage01 (version 2.0.3-4b1f869f0f) - partition WITHOUT quorum
   * Last updated: Tue Dec 27 07:42:45 2022
   * Last change:  Tue Dec 27 07:40:34 2022 by hacluster via crmd on storage01
   * 2 nodes configured
   * 0 resource instances configured
 Node List:
   * Online: [ storage01 ]
   * OFFLINE: [ storage02 ]

PCSD Status:
  storage01: Online
  storage02: Online
```

等待一会后，再次查看状态

```
root@storage01:~# pcs cluster status
Cluster Status:
 Cluster Summary:
   * Stack: corosync
   * Current DC: storage02eth (version 2.0.3-4b1f869f0f) - partition with quorum
   * Last updated: Wed Dec 28 02:25:19 2022
   * Last change:  Wed Dec 28 02:25:12 2022 by root via cibadmin on storage01eth
   * 2 nodes configured
   * 1 resource instance configured
 Node List:
   * Online: [ storage01 storage02 ]

PCSD Status:
  storage01: Online
  storage02: Online
```

我们可以开始添加浮动IP了（网卡名和label加起来不能超过15个字符，否则会报错）

```
pcs resource create mgmtd_ip ocf:heartbeat:IPaddr2 ip=10.141.161.19 cidr_netmask=32 nic=enp183s0f0 iflabel=mgm op monitor interval=30s
```

查看状态

```
root@storage01:~# pcs status
Cluster name: ha_cluster

WARNINGS:
No stonith devices and stonith-enabled is not false

Cluster Summary:
  * Stack: corosync
  * Current DC: storage02eth (version 2.0.3-4b1f869f0f) - partition with quorum
  * Last updated: Wed Dec 28 02:24:18 2022
  * Last change:  Wed Dec 28 02:21:25 2022 by root via cibadmin on storage01eth
  * 2 nodes configured
  * 1 resource instance configured

Node List:
  * Online: [ storage01 storage02 ]

Full List of Resources:
  * mgmtd_ip    (ocf::heartbeat:IPaddr2):        FAILED (Monitoring) [ storage02 storage01 ]

Failed Resource Actions:
  * mgmtd_ip_monitor_0 on storage02eth 'not configured' (6): call=5, status='complete', exitreason='Interface label [enp183s0f0:fl-mgm] exceeds maximum character limit of 15', last-rc-change='2022-12-28 02:21:25Z', queued=0ms, exec=44ms
  * mgmtd_ip_monitor_30000 on storage02eth 'not configured' (6): call=6, status='complete', exitreason='Interface label [enp183s0f0:fl-mgm] exceeds maximum character limit of 15', last-rc-change='2022-12-28 02:21:25Z', queued=0ms, exec=42ms
  * mgmtd_ip_monitor_0 on storage01eth 'not configured' (6): call=5, status='complete', exitreason='Interface label [enp183s0f0:fl-mgm] exceeds maximum character limit of 15', last-rc-change='2022-12-28 02:21:25Z', queued=0ms, exec=40ms

Daemon Status:
  corosync: active/enabled
  pacemaker: active/enabled
  pcsd: active/enabled
```

可能会提示失败，还缺少这一步

```
pcs property set stonith-enabled=false
```

重新查看状态

```
root@storage01:~# pcs status
Cluster name: ha_cluster
Cluster Summary:
  * Stack: corosync
  * Current DC: storage02eth (version 2.0.3-4b1f869f0f) - partition with quorum
  * Last updated: Wed Dec 28 02:39:45 2022
  * Last change:  Wed Dec 28 02:39:23 2022 by root via cibadmin on storage01eth
  * 2 nodes configured
  * 1 resource instance configured

Node List:
  * Online: [ storage01 storage02 ]

Full List of Resources:
  * mgmtd_ip    (ocf::heartbeat:IPaddr2):        Started storage01

Daemon Status:
  corosync: active/enabled
  pacemaker: active/enabled
  pcsd: active/enabled
```

接下来查看ip，应该浮动ip就会存在了