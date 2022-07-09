---
title: centos8安装ceph octopus集群
description: 
published: true
date: 2022-07-09T08:11:57.621Z
tags: install, centos8, ceph
editor: markdown
dateCreated: 2022-06-23T09:49:57.399Z
---

**简介：** centos8安装ceph octopus集群
## 集群详情
|        |               |      |
| ------ | ------------- | ---- |
| 集群名 | IP Address    |      |
| node01 | 10.141.160.50 |      |
| node02 | 10.141.160.51 |      |
| node03 | 10.141.160.52 |      |
## 安装

node01作为主要的host，先安装cephadm

### 三个节点都要安装的包

安装docker，由于centos8默认安装了podman，**安装docker会覆盖podman**

```
dnf config-manager --add-repo=https://download.docker.com/linux/centos/docker-ce.repo
dnf makecache
dnf install docker-ce -y --allowerasing
mkdir /etc/docker
cat > /etc/docker/daemon.json << EOF
{
  "registry-mirrors" : [
    "http://ovfftd6p.mirror.aliyuncs.com",
    "http://registry.docker-cn.com",
    "http://docker.mirrors.ustc.edu.cn",
    "http://hub-mirror.c.163.com"
  ],
  "insecure-registries" : [
    "registry.docker-cn.com",
    "docker.mirrors.ustc.edu.cn"
  ],
  "debug" : true,
  "experimental" : true
}
EOF
```

### node01初始化

安装依赖

```
dnf install -y python3 vim net-tools telnet
```

配置chrony

```
dnf install -y chrony
systemctl enable --now chronyd
```

关闭防火墙和selinux

```
systemctl disable --now firewalld
setenforce 0
sed -i s/SELINUX=enforcing/SELINUX=permissive/g /etc/selinux/config
```

安装cephadm

```
curl --silent --remote-name --location https://github.com/ceph/ceph/raw/octopus/src/cephadm/cephadm
chmod +x cephadm
```

安装ceph-common

```
./cephadm add-repo --release octopus
sed -i 's#download.ceph.com#mirrors.aliyun.com/ceph#' /etc/yum.repos.d/ceph.repo
dnf makecache
dnf install -y ceph-common
```

开始初始化

```
cephadm bootstrap --mon-ip 10.141.160.50
```

