---
title: cvmfs客户端配置
description: 
published: true
date: 2022-09-08T15:21:58.469Z
tags: ubuntu, centos, cvmfs
editor: markdown
dateCreated: 2022-09-08T15:21:58.469Z
---

### ubuntu

```
wget https://ecsft.cern.ch/dist/cvmfs/cvmfs-release/cvmfs-release-latest_all.deb
dpkg -i cvmfs-release-latest_all.deb
rm -f cvmfs-release-latest_all.deb
apt update
apt-get install cvmfs
```

### centos 

```
sudo yum install -y https://ecsft.cern.ch/dist/cvmfs/cvmfs-release/cvmfs-release-latest.noarch.rpm
sudo yum install -y cvmfs
```

从stratum0传输公钥到客户端上

```
scp /etc/cvmfs/keys/cvmfs.example.com.pub test@10.141.255.210:/home/test
```

创建客户端配置文件

```
cat > /etc/cvmfs/default.d/60-cvm-example-com.conf << EOF
CVMFS_CONFIG_REPOSITORY=cvmfs.example.com
CVMFS_DEFAULT_DOMAIN=example.com
CVMFS_SERVER_URL="http://172.16.0.1/cvmfs/cvmfs.example.com"
CVMFS_KEYS_DIR="/etc/cvmfs/keys/example.com/cvmfs.example.com.pub"
CVMFS_HTTP_PROXY=DIRECT
EOF
```

或者手动挂载

```
sudo mount -t cvmfs cvmfs.example.com /mnt
```

