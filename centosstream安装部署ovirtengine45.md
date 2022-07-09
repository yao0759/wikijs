---
title: centos stream安装部署ovirt engine 4.5
description: 
published: true
date: 2022-07-09T08:12:01.416Z
tags: install, ovirt, centos
editor: markdown
dateCreated: 2022-06-23T12:36:19.954Z
---

**简介：** centos stream安装部署ovirt engine 4.5

先要将package更新到最新

首次更新建议重启一下machine方便切换到新更新的内核版本，然后就可以安装启用ovirt源了。

```
dnf install -y centos-release-ovirt45
```

上述启用的是4.5版本的仓库源，假如需要安装ovirt4.4版本，可以执行

```
dnf install https://resources.ovirt.org/pub/yum-repo/ovirt-release44.rpm
```

接下来需要开启软件库

启用module **javapackages-tools** **pki-deps postgresql:12 mod\_auth\_openidc:2.3 nodejs:14**

```
dnf module -y enable javapackages-tools
dnf module -y enable pki-deps
dnf module -y enable postgresql:12
dnf module -y enable mod_auth_openidc:2.3
dnf module -y enable nodejs:14
```

然后同步已安装的软件包，让它们更新到最新版本

确保所有的packages都是最新

安装ovirt-engine（这个步骤需要花最多的时间）

开始安装初始化engine

在该命令执行过程中会有一些交互需要你选择是否启动哪些功能，一般默认选择即可，详细信息可以查看[https://www.ovirt.org/documentation/installing\_ovirt\_as\_a\_standalone\_manager\_with\_remote\_databases/](https://www.ovirt.org/documentation/installing_ovirt_as_a_standalone_manager_with_remote_databases/)中的3.4部分，里面有详细说明

安装完后，我们还需要重启一下服务

```
systemctl restart ovirt-engine
```

