---
title: Centos7安装wiki.js
description: 
published: true
date: 2022-07-09T08:11:26.622Z
tags: centos7, install, wikijs
editor: markdown
dateCreated: 2022-06-23T12:24:44.134Z
---

**简介：** Centos7安装wiki.js

## 3.\*特性前瞻

-   支持直接github同步  
    
-   更完善的用户管理系统  
    
-   页面关联功能增强可以更加容易配置文档之间的关联

## wiki.js特点

-   强大的编辑器  
    
-   预置了多种网站分析工具，如baidu,google,yandex  
    
-   支持标签功能  
    

## 安装node.js和wiki.js

### 安装依赖

```
yum groupinstall -y "Development"
yum install -y gcc-c++ make
```

### 安装node14

```
curl -sL https://rpm.nodesource.com/setup_14.x | sudo -E bash -
yum install -y nodejs
```

### 编译安装2.21版本的git

```
yum install -y gettext-devel openssl-devel perl-CPAN perl-devel zlib-devel curl-devel
cd /tmp
wget https://mirrors.edge.kernel.org/pub/software/scm/git/git-2.21.0.tar.gz && tar zxvf git-2.21.0.tar.gz
rm git-2.21.0.tar.gz
cd git-2.21.0/
./configure make prefix=/usr/local all
make prefix=/usr/local install
```

### 添加postgrelsql库并安装

```
yum install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-7-x86_64/pgdg-redhat-repo-latest.noarch.rpm
yum install postgresql13 postgresql13-server postgresql13-contrib
```

### 创建postgresql目录并启动

```
mkdir -p /data0/pgsql
chown -R postgres:postgres /data0/pgsql
sudo -u postgres /usr/pgsql-13/bin/initdb -D /data0/pgsql
mkdir -p /var/log/pgsql
chown -R postgres:postgres /var/log/pgsql
sudo -u postgres /usr/pgsql-13/bin/pg_ctl -D /data0/pgsql -l /var/log/pgsql/server.log start
```

### 下载安装wiki.js

```
cd /tmp
wget https://github.com/Requarks/wiki/releases/download/2.5.170/wiki-js.tar.gz
mkdir wiki
tar xzf wiki-js.tar.gz -C ./wiki
mv wiki /usr/local/
rm -rf wiki-js.tar.gz
cd /usr/local/wiki
mv config.sample.yml config.yml
```

### 修改config

```
vim config.yml
  
  bindIP: 127.0.0.1
  
  port: 3000
  
  db:
  type: postgres
  host: localhost
  port: 5432
  user: wikijs
  pass: wikijsrocks
  db: wiki
  
  dataPath: /data0/wiki
```

### 添加wiki服务

```
cat > /etc/systemd/system/wiki.service << EOF
[Unit]
Description=Wiki.js
After=network.target

[Service]
Type=simple
WorkingDirectory=/usr/local/wiki
ExecStart=/usr/bin/node server
Restart=always

User=root
Environment=NODE_ENV=production

[Install]
WantedBy=multi-user.target
EOF
systemctl daemon-reload
```

### 设置rc-local开机服务

```
cat > /etc/rc.local << EOF
#!/bin/bash











touch /var/lock/subsys/local
sudo -u postgres /usr/pgsql-13/bin/pg_ctl -D /data0/pgsql -l /var/log/pgsql/server.log start
EOF
chmod +x /etc/rc.local
```

```
cat > /etc/systemd/system/rc-local.service << EOF
[Unit]
Description=/etc/rc.d/rc.local Compatibility
ConditionFileIsExecutable=/etc/rc.d/rc.local
After=network.target

[Service]
Type=forking
ExecStart=/etc/rc.d/rc.local start
TimeoutSec=0
RemainAfterExit=yes
EOF
systemctl daemon-reload
```

## 编译安装nginx1.19.17

### 安装依赖

```
yum install -y epel-release
yum install -y perl perl-devel perl-ExtUtils-Embed libxslt libxslt-devel libxml2 libxml2-devel gd gd-devel GeoIP GeoIP-devel
yum install -y wget pcre-devel pcre zlib openssl openssl-devel
```

### 下载源码并编译

```
cd /root
git clone https://github.com/google/ngx_brotli.git
cd ngx_brotli/
git submodule update --init
```

```
cd /tmp
wget http://nginx.org/download/nginx-1.19.7.tar.gz
tar zxf nginx-1.19.7.tar.gz
cd nginx-1.19.7
./configure --prefix=/usr/share/nginx --sbin-path=/usr/sbin/nginx --modules-path=/usr/lib64/nginx/modules --conf-path=/etc/nginx/nginx.conf --error-log-path=/var/log/nginx/error.log --http-log-path=/var/log/nginx/access.log --http-client-body-temp-path=/var/lib/nginx/tmp/client_body --http-proxy-temp-path=/var/lib/nginx/tmp/proxy --http-fastcgi-temp-path=/var/lib/nginx/tmp/fastcgi --http-uwsgi-temp-path=/var/lib/nginx/tmp/uwsgi --http-scgi-temp-path=/var/lib/nginx/tmp/scgi --pid-path=/run/nginx.pid --lock-path=/run/lock/subsys/nginx --user=nginx --group=nginx --with-compat --with-file-aio --with-ipv6 --with-http_ssl_module --with-http_v2_module --with-http_realip_module --with-stream_ssl_preread_module --with-http_addition_module --with-http_xslt_module=dynamic --with-http_image_filter_module=dynamic --with-http_geoip_module=dynamic --with-http_sub_module --with-http_dav_module --with-http_flv_module --with-http_mp4_module --with-http_gunzip_module --with-http_gzip_static_module --with-http_random_index_module --with-http_secure_link_module --with-http_degradation_module --with-http_slice_module --with-http_stub_status_module --with-http_perl_module=dynamic --with-http_auth_request_module --with-mail=dynamic --with-mail_ssl_module --with-pcre --with-pcre-jit --with-stream=dynamic --with-stream_ssl_module --with-debug --add-module=/root/ngx_brotli
make && make install
```

### 添加nginx服务

```
cat > /usr/lib/systemd/system/nginx.service << EOF
[Unit]
Description=nginx - high performance web server
Documentation=https://nginx.org/en/docs/
After=network-online.target remote-fs.target nss-lookup.target
Wants=network-online.target

[Service]
Type=forking
PIDFile=/run/nginx.pid
ExecStartPre=/usr/sbin/nginx -t -c /etc/nginx/nginx.conf
ExecStart=/usr/sbin/nginx -c /etc/nginx/nginx.conf
ExecReload=/bin/kill -s HUP
ExecStop=/bin/kill -s TERM

[Install]
WantedBy=multi-user.target
EOF
systemctl daemon-reload
```

