---
title: cvmfs排查方法
description: 该文章具体参考https://cvmfs.readthedocs.io/en/stable/cpt-quickstart.html#troubleshooting
published: true
date: 2023-04-30T02:40:37.150Z
tags: cvmfs
editor: markdown
dateCreated: 2023-01-06T18:33:38.966Z
---

为了检查基本设置中的常见错误配置，运行

```
cvmfs_config chksetup
```
CernVM-FS从不同的配置文件中收集配置参数，这些文件可以覆盖彼此的设置（默认配置、域特定配置、本地设置...）。要显示repository.cern.ch的有效配置，请运行
```
cvmfs_config showconfig repository.cern.ch
```

为了排除autofs/automounter这个问题的来源，你可以尝试用以下方法手动加载 repository.cern.ch

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


为了排除SELinux是问题的来源，你可以在通过以下方式禁用SELinux后尝试安装

```
/usr/sbin/setenforce 0
```

一旦发现问题，确保通过重启autofs来进行修改

```
systemctl restart autofs
```
如果问题是一个版本库可以被挂载和卸载，但后来不能被重新挂载，请看重新挂载和命名空间/容器。

为了排除损坏的本地缓存是问题的来源，运行
```
cvmfs_config wipecache
```