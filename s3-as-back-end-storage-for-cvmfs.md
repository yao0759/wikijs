---
title: S3 as back-end storage for CVMFS
description: 
published: true
date: 2022-09-09T14:01:23.842Z
tags: install, cvmfs, s3
editor: markdown
dateCreated: 2022-09-09T14:00:56.558Z
---

## stratum 0安装

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
```

创建S3配置文件

```
CVMFS_S3_ACCOUNTS=1
CVMFS_S3_BUCKETS_PER_ACCOUNT=1
CVMFS_S3_ACCESS_KEY=[Access_key]
CVMFS_S3_SECRET_KEY=[secet_key]
CVMFS_S3_BUCKET=new-bucket-******
CVMFS_S3_HOST=s3.amazonaws.com
CVMFS_S3_PORT=80
CVMFS_S3_MAX_OF_PARALLEL_CONNECTIONS=400
```

aws这个访问密钥除了要读写上传权限，还需要ACL权限，不然会出现问题，权限策略可以参考下述配置

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "VisualEditor0",
            "Effect": "Allow",
            "Action": [
                "s3:ListBucket",
                "s3:GetBucketLocation"
            ],
            "Resource": "arn:aws:s3:::bucket"
        },
        {
            "Sid": "VisualEditor1",
            "Effect": "Allow",
            "Action": [
                "s3:PutObject",
                "s3:GetObject",
                "s3:DeleteObject",
                "s3:PutObjectAcl"
            ],
            "Resource": "arn:aws:s3:::bucket/*"
        }
    ]
}
```

执行例子

```
cvmfs_server mkfs  -u S3,/mnt,suffix@/etc/cvmfs/s3.conf -w http://********.s3.amazonaws.com/suffix  public.example.com
```

这里我执行的是

```
cvmfs_server mkfs  -u S3,/mnt,cvmfs@/etc/cvmfs/s3.conf -w http://********.s3.amazonaws.com/cvmfs  public.example.com
```



