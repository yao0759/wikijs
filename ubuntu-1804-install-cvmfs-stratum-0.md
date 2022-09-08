---
title: ubuntu 18.04安装cvmfs stratum 0
description: 
published: true
date: 2022-09-08T15:23:38.748Z
tags: install, ubuntu, cvmfs
editor: markdown
dateCreated: 2022-09-08T15:10:29.671Z
---

我这里使用的系统发行版本是ubuntu 18.04，所以stratum 0的安装步骤如下

下载安装源

```
wget https://ecsft.cern.ch/dist/cvmfs/cvmfs-release/cvmfs-release-latest_all.deb
dpkg -i cvmfs-release-latest_all.deb
rm -f cvmfs-release-latest_all.deb
```

安装

```
apt update
```

```
apt-get install cvmfs cvmfs-server
```

启用apache2

```
systemctl enable apache2
systemctl start apache2
```

开始配置，先定义仓库域名

```
MY_REPO_NAME=cvmfs.example.com
sudo systemctl restart autofs
sudo cvmfs_server mkfs -o $USER cvmfs.example.com
```

将仓库配置成事务状态，才能对其进行读写

```
cvmfs_server transaction ${MY_REPO_NAME}
```

完成后，执行下述命令

```
cvmfs_server publish ${MY_REPO_NAME}
```

每个cernvm-fs存储库都有一个白名单，其中包含对存储库签名的证书指纹。这个白名单的默认过期时间是30天，因此需要进场更新白名单。

如果主密钥保存在stratum0服务器上，则可以配置一个cronjob重新配置白名单。

```
vim /etc/cron.d/cvmfs_resign
...
0 11 * * 1 root /usr/bin/cvmfs_server resign repo.organization.tld
...
```

还建议增大一下最大文件和chunk大小，默认只有1024MB

```
# Created by cvmfs_server.
CVMFS_CREATOR_VERSION=143
CVMFS_REPOSITORY_NAME=cvmfs.example.com
CVMFS_REPOSITORY_TYPE=stratum0
CVMFS_USER=spadm
CVMFS_UNION_DIR=/cvmfs/cvmfs.example.com
CVMFS_SPOOL_DIR=/var/spool/cvmfs/cvmfs.example.com
CVMFS_STRATUM0=http://localhost/cvmfs/cvmfs.example.com
CVMFS_UPSTREAM_STORAGE=local,/srv/cvmfs/cvmfs.example.com/data/txn,/srv/cvmfs/cvmfs.example.com
CVMFS_USE_FILE_CHUNKING=true
CVMFS_MIN_CHUNK_SIZE=4194304
CVMFS_AVG_CHUNK_SIZE=8388608
CVMFS_MAX_CHUNK_SIZE=16777216
CVMFS_UNION_FS_TYPE=overlayfs
CVMFS_HASH_ALGORITHM=sha1
CVMFS_COMPRESSION_ALGORITHM=default
CVMFS_EXTERNAL_DATA=false
CVMFS_AUTO_TAG=true
CVMFS_AUTO_TAG_TIMESPAN=""
CVMFS_GARBAGE_COLLECTION=false
CVMFS_AUTO_REPAIR_MOUNTPOINT=true
CVMFS_AUTOCATALOGS=true
CVMFS_ASYNC_SCRATCH_CLEANUP=true
CVMFS_PRINT_STATISTICS=false
CVMFS_UPLOAD_STATS_DB=false
CVMFS_UPLOAD_STATS_PLOTS=false
CVMFS_IGNORE_XDIR_HARDLINKS=true
CVMFS_FILE_MBYTE_LIMIT=8192
CVMFS_AUTOCATALOGS_MAX_WEIGHT=1000000
CVMFS_AUTOCATALOGS_MIN_WEIGHT=1000
CVMFS_ROOT_KCATALOG_LIMIT=800000
```