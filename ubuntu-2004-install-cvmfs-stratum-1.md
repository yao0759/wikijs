---
title: ubuntu 20.04安装cvmfs stratum 1
description: 
published: true
date: 2022-09-08T15:14:46.052Z
tags: install, ubuntu, cvmfs
editor: markdown
dateCreated: 2022-09-08T15:13:57.378Z
---

**我这里使用的系统发行版本是ubuntu 20.04，所以stratum 1的安装步骤如下**

```
wget https://ecsft.cern.ch/dist/cvmfs/cvmfs-release/cvmfs-release-latest_all.deb
dpkg -i cvmfs-release-latest_all.deb
rm -f cvmfs-release-latest_all.deb
```

**安装**

```
apt update
```

```
apt-get install cvmfs cvmfs-server squid libapache2-mod-wsgi
```

**修改apche默认端口，将`Listen 80`修改成`Listen 127.0.0.1:8080`**

```
vim /etc/apache2/ports.conf
...
Listen 127.0.0.1:8080
...
```

**修改squid配置文件，将配置文件内容替换为下述部分。**

```
cat > /etc/squid/squid.conf << EOF
http_port 80 accel
http_port 8000 accel
http_access allow all
cache_peer 127.0.0.1 parent 8080 0 no-query originserver

acl CVMFSAPI urlpath_regex ^/cvmfs/[^/]*/api/
cache deny !CVMFSAPI

cache_mem 128 MB
EOF
```

**启动服务**

```
systemctl enable --now apache2
systemctl enable --now squid
```

接下来一步是配置DNS解析缓存，由于我的stratum1跟client在同一网段下，无需使用DNS解析缓存，这里我直接跳过，具体步骤可以参考这个链接[3. Stratum 1 and proxies - CernVM-FS tutorial (cvmfs-contrib.github.io)](https://cvmfs-contrib.github.io/cvmfs-tutorial-2021/03_stratum1_proxies/)

**同步传输密钥**

```
mkdir /etc/cvmfs/keys/cvmfs.example.com/
```

```
mv /home/spadm/cvmfs.example.com.pub /etc/cvmfs/keys/cvmfs.example.com/
```

**创建副本**

```
cvmfs_server add-replica -o $USER http://132.249.229.209/cvmfs/cvmfs.example.com /etc/cvmfs/keys/cvmfs.example.com/
```

创建定时更新脚本

```
cvmfs_server snapshot -a -a /etc/logrotate.d/cvmfs

...
/var/log/cvmfs/*.log {
    weekly
    missingok
    notifempty
}
...
```

```
cat > /etc/cron.d/cvmfs_stratum1_snapshot << EOF
*/5 * * * * root output=$(/usr/bin/cvmfs_server snapshot -a -i 2>&1) || echo "$output"
EOF
```
