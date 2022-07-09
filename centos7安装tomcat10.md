---
title: centos7安装tomcat10
description: 
published: true
date: 2022-07-09T08:11:53.897Z
tags: centos7, tomcat, install
editor: markdown
dateCreated: 2022-06-23T06:44:11.721Z
---

# centos7安装tomcat10

## tomcat 10 特性

tomcat10.0.x版本实现了**Servlet 5.0**, **JSP 3.0**, **EL 4.0**, **WebSocket 2.0** ，**Authentication 2.0**

## 安装步骤

关闭selinux

```shell
setenforce 0
sed -i 's/ELINUX=enforcing/ELINUX=disabled/g' /etc/selinux/config
```

tomcat10后，只支持安装openjdk11及以上版本

```shell
yum install java-11-openjdk-devel
```

安装完成后，查看java版本

```shell
[root@localhost ~]# java --version
openjdk 11.0.12 2021-07-20 LTS
OpenJDK Runtime Environment 18.9 (build 11.0.12+7-LTS)
OpenJDK 64-Bit Server VM 18.9 (build 11.0.12+7-LTS, mixed mode, sharing)
```

假如是其他版本，则需要使用下面的命令切换版本

```shell
alternatives --config java
```

新建tomcat用户

```shell
useradd -d /opt/tomcat -s /bin/nologin tomcat
```

下载tomcat10包

```shell
yum install -y wget
wget https://dlcdn.apache.org/tomcat/tomcat-10/v10.0.10/bin/apache-tomcat-10.0.10.tar.gz
tar zxf apache-tomcat-10.0.10.tar.gz
cd apache-tomcat-10.0.10
cp -r * /opt/tomcat
```

赋予tomcat权限

```shell
chown -R tomcat:tomcat /opt/tomcat/
```

添加systemd服务，配置服务需要修改**JAVA\_HOME**变量，可以通过**alternatives --list | grep ^java**查看

```shell
vi /etc/systemd/system/tomcat.service

Unit]
Description=Apache Tomcat Web Application Container
Wants=network.target
After=network.target

[Service]
Type=forking
Environment=JAVA_HOME=/usr/lib/jvm/java-11-openjdk-11.0.12.0.7-0.el7_9.x86_64
Environment=CATALINA_PID=/opt/tomcat/temp/tomcat.pid
Environment=CATALINA_HOME=/opt/tomcat
Environment='CATALINA_OPTS=-Xms512M -Xmx1G -Djava.net.preferIPv4Stack=true'
Environment='JAVA_OPTS=-Djava.awt.headless=true'
ExecStart=/opt/tomcat/bin/startup.sh
ExecStop=/opt/tomcat/bin/shutdown.sh
SuccessExitStatus=143
User=tomcat
Group=tomcat
UMask=0007
RestartSec=10
Restart=always

[Install]
WantedBy=multi-user.target
```

启用服务

```shell
systemctl start tomcat
systemctl enable tomcat
```

开启防火墙

```shell
firewall-cmd --permanent --add-port=8080/tcp
firewall-cmd --reload
```

配置tomcat admin-gui和manager-gui的认证

```shell
vi /usr/share/tomcat/conf/tomcat-users.xml
......
  <role rolename="admin-gui"/>
  <role rolename="manager-gui"/>
  <user username="tomcat" password="tomcat" roles="manager-gui,admin-gui"/>
</tomcat-users>
```