---
title: ubuntu20.04安装guacamole server 1.3
description: 
published: true
date: 2022-07-09T08:13:02.704Z
tags: install, ubuntu, guacamole
editor: markdown
dateCreated: 2022-06-23T10:07:43.144Z
---

2022-04-29 55

**简介：** ubuntu20.04安装guacamole server 1.3

1.  更新package

```
apt update 
apt upgrade -y
```

2.  安装依赖包

```
apt install build-essential libcairo2-dev libjpeg-turbo8-dev \
    libpng-dev libtool-bin libossp-uuid-dev libvncserver-dev \
    freerdp2-dev libssh2-1-dev libtelnet-dev libwebsockets-dev \
    libpulse-dev libvorbis-dev libwebp-dev libssl-dev \
    libpango1.0-dev libswscale-dev libavcodec-dev libavutil-dev \
    libavformat-dev
```

3.  下载guacamole 1.3 源码包

```
cd /opt
wget https://downloads.apache.org/guacamole/1.3.0/source/guacamole-server-1.3.0.tar.gz
```

4.  解压文件

```
tar -xvf guacamole-server-1.3.0.tar.gz
cd guacamole-server-1.3.0
```

5.  开始编译

```
./configure --with-init-dir=/etc/init.d --enable-allow-freerdp-snapshots
make
make install
```

6.  更新lib并重新加载systemd服务

```
ldconfig
systemctl daemon-reload
```

7.  启动服务

```
systemctl start guacd
systemctl enable guacd
```

8.  创建一个目录来存放guacamole配置文件和扩展。

```
mkdir -p /etc/guacamole/{extensions,lib}
```

9.  **开始安装guacamole web app**。需要先安装apache tomcat

```
apt install tomcat9 tomcat9-admin tomcat9-common tomcat9-user
```

10.  下载guacamole client

```
wget https://downloads.apache.org/guacamole/1.3.0/binary/guacamole-1.3.0.war
```

11.  拷贝客户端到tomcat web目录

```
mv guacamole-1.3.0.war /var/lib/tomcat9/webapps/guacamole.war
```

12.  重启tomcat和guacd服务

```
systemctl restart tomcat9 guacd
```

13.  **开始配置数据库。**先安装mariadb

```
apt install mariadb-server -y
```

14.  初始化数据库

```
mysql_secure_installation
```

15.  初始化后，在连接前，需要安装mysql connector和guacamole jdbc插件

```
wget https://dev.mysql.com/get/Downloads/Connector-J/mysql-connector-java-8.0.26.tar.gz
tar -xf mysql-connector-java-8.0.26.tar.gz
cp mysql-connector-java-8.0.26/mysql-connector-java-8.0.26.jar /etc/guacamole/lib/
```

16.  下载jdbc身份验证插件

```
wget https://downloads.apache.org/guacamole/1.3.0/binary/guacamole-auth-jdbc-1.3.0.tar.gz
tar -xf guacamole-auth-jdbc-1.3.0.tar.gz
mv guacamole-auth-jdbc-1.3.0/mysql/guacamole-auth-jdbc-mysql-1.3.0.jar /etc/guacamole/extensions/
```

17.  登录mysql，创建数据库

18.  完成后退出，到jdbc插件目录下的schema目录里

```
cd guacamole-auth-jdbc-1.3.0/mysql/schema
```

19.  导入sql schema文件到数据库

```
cat *.sql | mysql -u root -p guacamole_db
```

20.  创建properties文件

```
vim /etc/guacamole/guacamole.properties
```

添加下述信息到文件内

```

mysql-hostname: 127.0.0.1
mysql-port: 3306
mysql-database: guacamole_db
mysql-username: guacamole_user
mysql-password: [password]
```

21.  重启服务

```
systemctl restart tomcat9 guacd mysql
```

22.  访问

http://<ip>:8080/guacamole