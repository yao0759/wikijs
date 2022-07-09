---
title: ubuntu20.04编译安装nginx 1.21.6并开启fastopen
description: 
published: true
date: 2022-07-09T08:13:06.202Z
tags: install, ubuntu, nginx, fastopen
editor: markdown
dateCreated: 2022-06-23T09:56:10.345Z
---

**简介：** ubuntu20.04编译安装nginx 1.21.6并开启fastopen

先安装依赖

```
apt install -y perl libperl-dev libgd3 libgd-dev libgeoip1 libgeoip-dev geoip-bin libxml2 libxml2-dev libxslt1.1 libxslt1-dev libssl-dev wget gnupg2 ca-certificates   
```

下载源码

```
wget http://nginx.org/download/nginx-1.21.6.tar.gz
```

解压

```
tar zxf nginx-1.21.6.tar.gz -C /opt
```

准备编译

需要开启tcp fast open需要在--with-cc-opt=里加入-DTCP\_FASTOPEN=23

```
./configure --prefix=/etc/nginx --sbin-path=/usr/sbin/nginx --modules-path=/usr/lib/nginx/modules --conf-path=/etc/nginx/nginx.conf --error-log-path=/var/log/nginx/error.log --http-log-path=/var/log/nginx/access.log --pid-path=/var/run/nginx.pid --lock-path=/var/run/nginx.lock --http-client-body-temp-path=/var/cache/nginx/client_temp --http-proxy-temp-path=/var/cache/nginx/proxy_temp --http-fastcgi-temp-path=/var/cache/nginx/fastcgi_temp --http-uwsgi-temp-path=/var/cache/nginx/uwsgi_temp --http-scgi-temp-path=/var/cache/nginx/scgi_temp --user=nginx --group=nginx --with-compat --with-file-aio --with-threads --with-http_addition_module --with-http_auth_request_module --with-http_dav_module --with-http_flv_module --with-http_gunzip_module --with-http_gzip_static_module --with-http_mp4_module --with-http_random_index_module --with-http_realip_module --with-http_secure_link_module --with-http_slice_module --with-http_ssl_module --with-http_stub_status_module --with-http_sub_module --with-http_v2_module --with-mail --with-mail_ssl_module --with-stream --with-stream_realip_module --with-stream_ssl_module --with-stream_ssl_preread_module --with-cc-opt='-g -O2 -fdebug-prefix-map=/data/builder/debuild/nginx-1.20.2/debian/debuild-base/nginx-1.20.2=. -DTCP_FASTOPEN=23  -fstack-protector-strong -Wformat -Werror=format-security -Wp,-D_FORTIFY_SOURCE=2 -fPIC' --with-ld-opt='-Wl,-Bsymbolic-functions -Wl,-z,relro -Wl,-z,now -Wl,--as-needed -pie'
```
