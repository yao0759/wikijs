---
title: minio配置tls以开启https访问
description: 
published: true
date: 2022-07-09T08:12:32.277Z
tags: minio, https
editor: markdown
dateCreated: 2022-06-23T09:26:57.957Z
---

**简介：** minio配置tls以开启https访问

minio默认启用http作为连接方式，假如需要开启https的话，需要通过openssl命令生成证书，不建议使用certgen命令生成自签证书，因为会有些错误，即便官方文档中有提到certgen这种方式。

1\. 创建openssl.conf

```
[req]
   distinguished_name = req_distinguished_name
   x509_extensions = v3_req
   prompt = no
   
   [req_distinguished_name]
   C = US
   ST = VA
   L = Somewhere
   O = MyOrg
   OU = MyOU
   CN = MyServerName
   
   [v3_req]
   subjectAltName = @alt_names
   
[alt_names]
   IP.1 = 127.0.0.1
   DNS.1 = localhost
```

2\. 生成证书

```
cd /home/.minio/certs
openssl req -new -x509 -nodes -days 730 -keyout private.key -out public.crt -config openssl.conf
```

3\. 重启服务或者重新启动进程

