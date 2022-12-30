---
title: 从源码安装iozone
description: 
published: true
date: 2022-12-30T15:34:16.103Z
tags: iozone
editor: markdown
dateCreated: 2022-12-30T15:34:16.103Z
---

不同的系统版本提供的iozone版本不一致，由于要对比不同文件系统的性能差异，最好iozone的版本保持一致，所以需要编译安装iozone



1. 下载源码

```
wget http://www.iozone.org/src/current/iozone3_492.tgz
```

1. 解压

```
tar zxf iozone3_492.tgz
```

1. 编译

```
cd iozone3_492/src/current/
make linux-AMD64
```

在编译安装时，依赖automake1.14，假如需要了可能需要编译这个版本的automake才行