---
title: 如何编译安装最新版本orangefs
description: 
published: true
date: 2023-10-06T11:47:11.559Z
tags: 
editor: markdown
dateCreated: 2023-10-06T11:47:11.559Z
---

## orangefs 2.10.0版本特性

orangefs 2.10.0添加了不少功能特性，作为pvfs的分支之一，orangefs是一个优秀的值得研究的并行文件系统，根据[链接]( http://download.orangefs.org/current/source/ChangeLog.txt) 
可以知道这次的版本更新带了以下特性：

- general updates

  1. numerous autotools related m4 files are updated.
  2. code refactored in numerous places to eliminate compiler warnings from new versions of gcc.
  3. documentation updates.

- Server
  1. updated formatting to conform to standard in simple-stripe code.

- Clients
  1. build liborangefs by default. liborangefs provides an orangefs version of posix system calls.

  2. Windows Client
         - Retargeted Client to the user mode file system library "dokany" v1.5.1.1000. (See https://github.com/dokan-dev/dokany)
         - Retargeted build to Visual Studio 2022.
         - Updated Event Logging for modern versions of Windows.
         - Removed obsolete security modes.
         - Updated the executable installer to install dokany and necessary Visual C++ runtime libraries.
         - Updated OpenSSL.
         - Made several quality-of-life improvements.

- bmi

  1. Eliminated IBV_EVENT_WQ_FATAL warnings
  2. Added a new RDMA module:
     -  Utilizes the RDMA Communication Manager library (librdmacm), abstracting transport-specific details out of connection management
     -  Adds RoCE v2 support to BMI
  3. Added a new experimental InfiniBand module for future development:

     - Still a work in progress and minimally tested
     - Current experimental changes include:
     - Improved connection-based reference counting

## 安装过程

这里我使用的环境是fedora38，为什么使用这样一个系统版本，主要原因是在查看官方文档时，发现它对红帽系的支持更好，且2.10.0版本中，5.15.0以上的内核版本似乎提供更好的性能，所以我用了一个较新的系统版本，在生产环境中，应该更推荐almalinux9或者rocky9
我们先安装解决一些包的依赖问题：

```
sudo dnf update -y
```

```
sudo yum -y install gcc flex bison openssl-devel libattr-devel kernel-devel perl make automake
```

```
sudo dnf install -y "Development Tools"

```

然后下载2.10.0 release源码包

```
wget https://github.com/waltligon/orangefs/releases/download/v.2.10.0/orangefs-2.10.0.tar.gz
```

准备开始编译，先解压

```
tar zxf orangefs-2.10.0.tar.gz
cd orangefs
```

生成configure并检查缺少的包

```
./prepare
```

执行configure

```
./configure --prefix=/opt/orangefs --with-kernel=/lib/modules/6.5.5-200.fc38.x86_64/build/ --with-db-backend=lmdb --enable-shared --enable-epoll --enable-racache --enable-ucache --enable-visual
```

到这一步应该会出现下述报错：

```
configure: error: The kernel source tree does not appear to be 2.6 or 3.X or 4.X
```

这个问题我怀疑是configure里面还是指定需要4.*以下的内核版本，我们进去把这部分注释掉，如下述图片这样

![image-20231006194245246](https://yhblog-1254039996.cos.ap-guangzhou.myqcloud.com/img-blog/image-20231006194245246.png)


安装

```
make & make install
```