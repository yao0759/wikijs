---
title: centos7从源码编译openzfs 2.1 rpm
description: 
published: true
date: 2022-07-09T08:11:49.872Z
tags: centos7, install, openzfs
editor: markdown
dateCreated: 2022-06-23T06:56:37.472Z
---

## zfs2.1新特性

-   dRAID，该项功能，能够加快磁盘重建速度
-   Compatibility Property：能够在zpool启用该功能，保证openzfs不同版本和不同平台的兼容性
-   同时支持了influxdb

## 安装步骤

先安装依赖包

```
yum install epel-release
yum groupinstall -y "Development"
yum update -y
yum install autoconf automake gettext createrepo libuuid-devel libblkid-devel openssl-devel libtirpc-devel lz4-devel libzstd-devel zlib-devel kernel-devel elfutils-libelf-devel libaio-devel libattr-devel libudev-devel python3-devel libffi-devel ncompress python2-devel python-cffi python-setuptools wget
```

下载源码，目前最新2.1稳定版本已经释放了，所以我测试的是2.1版本

```
cd /opt
wget https://github.com/openzfs/zfs/archive/refs/tags/zfs-2.1.0.tar.gz
tar zxf zfs-2.1.0.tar.gz
./autogen.sh
./configure
make rpm
```

编译成功后，创建本地库并安装

```
cat > /etc/yum.repos.d/zfs-local.repo << EOF
[zfs-local]
name=ZFS Local
baseurl=file:///var/lib/zfs.repo
enabled=1
gpgcheck=0
EOF
mkdir -p /var/lib/zfs.repo
createrepo /var/lib/zfs.repo
cp *.rpm /var/lib/zfs.repo/
createrepo --update /var/lib/zfs.repo
yum --enablerepo=zfs-local install zfs
/sbin/modprobe zfs
lsmod | grep zfs
```