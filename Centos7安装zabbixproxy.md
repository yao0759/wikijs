---
title: Centos7安装zabbix proxy
description: 
published: true
date: 2022-07-09T08:11:30.450Z
tags: centos7, install, zabbix
editor: markdown
dateCreated: 2022-06-23T10:06:45.132Z
---

2022-04-29 31

**简介：** Centos7安装zabbix proxy

## 为什么需要安装zabbix proxy？

因为zabbix server性能羸弱，无法支撑太多的server和item。因为加入一个proxy减轻server的压力

## 安装

1.  先安装mariadb  
    

```
yum install -y mariadb-server mariadb
systemctl start mariadb.service
systemctl enable mariadb.service
mysql_secure_installation
```

2.  创建zabbix\_proxy数据库  
    

```
mysql -uroot -p
mysql> create database zabbix_proxy character set utf8 collate utf8_bin;
mysql> grant all privileges on zabbix_proxy.* to zabbix@localhost identified by 'zabbix';
mysql> quit;
```

3.  安装zabbix-proxy-mysql并导入数据库  
    

```
yum install https://repo.zabbix.com/zabbix/5.0/rhel/7/x86_64/zabbix-release-5.0-1.el7.noarch.rpm
yum install -y zabbix-proxy-mysql
zcat /usr/share/doc/zabbix-proxy-mysql*/schema.sql.gz | mysql -uzabbix -p zabbix_proxy
```

4.  配置/etc/zabbix/zabbix\_proxy.conf  
    

```
Server=************
Hostname=zabbix_proxy
DBName=zabbix_proxy
DBUser=zabbix
DBPassword=zabbix
```

5.  到zabbix server webui上添加proxy（agent代理名称必须跟/etc/zabbix/zabbix\_proxy.conf中配置hostname一致）  
    

![image-20220411170026645.png](https://ucc.alicdn.com/pic/developer-ecology/0d74c5788d404b8baf66ff7b750e9e59.png "image-20220411170026645.png")

![image-20220411170043851.png](https://ucc.alicdn.com/pic/developer-ecology/b99ff87f1f274a13a496b0058abca2f4.png "image-20220411170043851.png")

