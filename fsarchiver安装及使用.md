---
title: fsarchiver安装及使用
description: 
published: true
date: 2022-09-04T09:40:47.691Z
tags: fsarchiver, backup
editor: markdown
dateCreated: 2022-06-23T09:25:47.942Z
---

## 特性

-   支持基本的文件属性（权限、所有权...）。  
    
-   支持所有Linux文件系统的基本文件系统属性（Label、uuid、块大小）。  
    
-   支持每个存档的多个文件系统  
    
-   支持扩展的文件属性（SELinux使用）。  
    
-   支持所有主要的Linux文件系统（extfs、xfs、btrfs、reiserfs等）。  
    
-   支持FAT文件系统（以便备份/恢复EFI系统分区）  
    
-   实验性地支持克隆ntfs文件系统  
    
-   对写在存档中的所有内容进行校验（标题、数据块、整个文件）。  
    
-   能够恢复已损坏的存档（它将跳过当前文件）  
    
-   多线程lzo、gzip、bzip2、lzma/xz压缩：如果你有一个双核/四核，它将使用你的cpu的所有能力。  
    
-   Lzma/xz压缩（缓慢但非常有效的算法），使压缩文件更小。  
    
-   支持将大档案分割成几个有固定最大尺寸的文件  
    
-   使用密码对档案进行加密。基于libgcrypt的blowfish。  
    

## 如何防止数据丢失

FSArchiver使用两级校验来保护你的数据免受损坏。每个文件的每个块都有一个写在存档中的32位校验和。这样我们就可以识别你的文件的哪个块被损坏了。一旦一个文件被恢复，整个文件的md5校验和将与原始md5进行比较。这是一个128位的校验和，所以它可以检测到所有的文件损坏。如果有一个文件被损坏，FSArchiver会恢复存档中的所有其他文件，所以你不会丢失所有的数据。这与tar.gz非常不同，在那里整个tar是用gzip压缩的。在这种情况下，损坏后写入的数据会丢失。

## 安装

你可以下载一个带有FSArchiver的 live cd（例如：任何最近的SystemRescue），或者你可以在你现有的Linux系统上安装它。如果你想安装它，你有三种解决方案。

-   使用你的Linux发行版附带的官方软件包（推荐方法）  
    
-   使用官方的静态二进制文件（非常简单：你只有一个独立的二进制文件需要复制）  
    
-   从源代码编译fsarchiver  
    

### 从源码编译

如果你想从源码编译，你将需要在你的系统上安装以下库，包括它们的头文件。在基于RPM的发行版上，你通常需要安装诸如libXXX-devel这样的包来获得头文件，因为基础包（libXXX）并不提供这些文件。下面是所需的库。

-   zlib (and zlib-devel)  
    
-   liblzo (and lzo-devel)  
    
-   lz4 (and lz4-devel)  
    
-   bzip2-libs (and bzip2-devel)  
    
-   libgcrypt (and libgcrypt-devel)  
    
-   libblkid (and libblkid-devel)  
    
-   e2fsprogs-libs (e2fsprogs-devel)  
    
-   [xz-utils](http://tukaani.org/xz/) (and xz-devel)  
    
-   [zstd](https://facebook.github.io/zstd/) (and libzstd-devel)  
    

你可以有选择地禁用对缺少的库或头文件的支持。例如，在一个缺少lzo和zstd的系统上。

./configure --prefix=/usr --disable-lzo --disable-zstd && make && make install

你也可以使用静态二进制文件，它提供对所有压缩级别的支持，并且不需要在你的系统上安装库。

要编译源代码，你必须按照这些说明运行。

#### 下载最新源码包

首先，你必须下载[fsarchiver-0.8.6.tar.gz](https://github.com/fdupoux/fsarchiver/releases/download/0.8.6/fsarchiver-0.8.6.tar.gz)。把它保存到一个临时目录中。

#### 在临时目录中提取源代码

```
mkdir -p /var/tmp/fsarchiver
cd /var/tmp/fsarchiver
tar xfz /path/to/fsarchiver-x.y.z.tar.gz
```

#### 从源码编译和安装

```
cd /var/tmp/fsarchiver/fsarchiver-x.y.z
./configure --prefix=/usr
make && make install
```

### 安装预编译的二进制文件

如果编译不成功，你可以直接下载静态二进制文件，将其解压，并将二进制文件复制到你系统的某个地方（例如/usr/sbin）。你可以使用32位版本（fsarchiver-static-x.y.z.i386.tar.gz）或者64位版本（fsarchiver-static-x.y.z.x86\_64.tar.gz）。

卸载fsarchiver很容易，因为它只在你的系统中安装一个文件，也就是编译后的二进制文件。要卸载fsarchiver，只需删除/usr/sbin/fsarchiver。

#### 依赖

FSArchiver 有两种依赖关系：库和文件系统工具。

#### 库

几个库和它们的头文件对于编译源文件（参考前一节关于安装的内容）和执行程序（如果没有以静态方式编译）是必要的。你可以通过使用静态二进制文件来避免库的依赖性问题，你可以从github发布页面下载。

#### 文件系统工具

要恢复一个文件系统，你所使用的文件系统的工具是必需的。当你保存文件系统（使用fsarchiver savefs）时，你不应该有任何关于文件系统工具的问题（比如一个程序因为没有安装而丢失）。只有当你想恢复一个文件系统时才需要文件系统工具（如e2fsprogs, reiserfsprogs, xfsprogs, ...）。这其实不是什么问题，因为你经常想通过Livecd（如SystemRescue）来恢复文件系统，因为你在使用根文件系统时无法恢复，所以在这种情况下必须从Livecd/U盘启动。

FSArchiver只需要与恢复的文件系统匹配的工具。例如，如果你试图将归档文件恢复到 reiserfs 分区，你将需要 reiserfsprogs 可用，即使在你将文件系统保存为归档文件时，原始文件系统是 ext3。

### 发行版的具体信息

使用你的发行版软件包管理器可以确保所有必要的依赖项都被自动安装。

#### fedora安装

#### ubuntu安装

```
sudo apt-get update
sudo apt-get install fsarchiver
```

#### debian安装

```
sudo apt-get update
sudo apt-get install fsarchiver
```

#### centos7.x安装

```
yum install https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm
yum install https://github.com/fdupoux/fsarchiver/releases/download/0.8.6/fsarchiver-0.8.6-1.el7.x86_64.rpm
```

假如需要编译安装

```
yum install zlib-devel bzip2-devel lzo-devel lz4-devel xz-devel libzstd-devel e2fsprogs-devel libgcrypt-devel libattr-devel libblkid-devel
tar xfz fsarchiver-0.8.6.tar.gz
cd fsarchiver-0.8.6
./configure --prefix=/usr && make && make install
```

## 测试例子

先将文件系统保存归档

```
fsarchiver savefs -o /backup/backup-fsa/backup-fsa025-gentoo-amd64-20090103-01.fsa /dev/sda1 /dev/sda2 -v -j4 -A
filesystem features: [has_journal,resize_inode,dir_index,filetype,sparse_super,large_file]
============= archiving filesystem /dev/sda1 =============
-[00][REGFILE ] /vmlinuz-2.6.25.20-x64-fd13
-[00][REGFILE ] /sysresccd/memdisk
-[00][REGFILE ] /sysresccd/pxelinux.0
-[00][REGFILE ] /sysresccd/initram.igz
-[00][REGFILE ] /sysresccd/boot.cat
.....
-[00][DIR     ] /mkbootcd-gentoo64
-[00][REGFILE ] /System.map-2.6.25.20-x64-fd13
-[00][REGFILE ] /config-2.6.25.20-x64-fd13
-[00][REGFILE ] /config-2.6.27.09-x64-fd16
-[00][DIR     ] /
============= archiving filesystem /dev/sda2 =============
-[01][SYMLINK ] /bin/bb
-[01][REGFILE ] /bin/dd
-[01][REGFILE ] /bin/cp
-[01][REGFILE ] /bin/df
.....
-[01][REGFILE ] /fdoverlay/profiles/repo_name
-[01][DIR     ] /fdoverlay/profiles
-[01][DIR     ] /fdoverlay
-[01][DIR     ] /
```

还原

```
fsarchiver restfs /backup/backup-fsa/backup-fsa025-gentoo-amd64-20090103-01.fsa id=0,dest=/dev/sda1 id=1,dest=/dev/sdb1
```