---
title: cvmfs排查方法
description: 
published: false
date: 2023-01-06T18:33:38.966Z
tags: cvmfs
editor: markdown
dateCreated: 2023-01-06T18:33:38.966Z
---

In order to check for common misconfigurations in the base setup, run

```
cvmfs_config chksetup
```
CernVM-FS gathers its configuration parameter from various configuration files that can overwrite each others settings (default configuration, domain specific configuration, local setup, …). To show the effective configuration for repository.cern.ch, run
```
cvmfs_config showconfig repository.cern.ch
```
In order to exclude autofs/automounter as a source of problems, you can try to mount repository.cern.ch manually with the following
```
mkdir -p /mnt/cvmfs
mount -t cvmfs repository.cern.ch /mnt/cvmfs
```

cvmfs的排查问题流程需要先去区分是客户端还是服务端的问题，还是就是需要结合报错信息去找到可能的原因。
## 客户端
假如是首次登陆挂载cvmfs的话，可以检查一下配置文件，通过`cvmfs_config`命令检查配置文件是否有问题。
```
cvmfs_config chksetup
```
假如是redhat发行版本，还可能因为selinux导致挂载问题，所以在测试期间，可以选择关闭临时关闭selinux
```
setenforce 0
```
假如stratum 0或者1，没有设置GeoIP，无法通过地理位置访问最近的网络，是直接访问到stratum0或者选择了较远的stratum1上，可能会因为网络延迟的原因，导致timeout，可以通过`journalctl`查看日志信息
```
journalctl -xe 
```



## 服务端


In order to exclude SELinux as a source of problems, you can try mounting after SELinux has been disabled by
```
/usr/sbin/setenforce 0
```
Once the issue has been identified, ensure that the changes are taken by restarting autofs
```
systemctl restart autofs
```
If the problem is that a repository can be mounted and unmounted but later cannot be remounted, see Remounting and Namespaces/Containers.

In order to exclude a corrupted local cache as a source of problems, run
```
cvmfs_config wipecache
```