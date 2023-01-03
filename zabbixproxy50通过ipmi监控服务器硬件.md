---
title: zabbix proxy 5.0通过ipmi监控服务器硬件
description: 
published: true
date: 2023-01-03T08:44:46.993Z
tags: 硬件, zabbix, ipmi
editor: markdown
dateCreated: 2022-06-23T12:38:00.085Z
---

**简介：** zabbix proxy 5.0通过ipmi监控服务器硬件

日常有不少的硬件服务器需要维护，除了添加系统监控外，建议通过snmp或者ipmi的方式监控硬件信息。

由于这里我主要是通过zabbix\_proxy监控ipmi，所以先在zabbix proxy上安装依赖包

```
yum install -y OpenIPMI OpenIPMI-devel ipmitool freeipmi
```

然后我们还需要开启ipmi收集线程，默认是不开启的，我这里无论是zabbix server还是zabbix proxy都开启了，修改/etc/zabbix/zabbix\_proxy.conf，取消StartIPMIPollers这行的注释，值可以先按默认的来，后面根据服务器规模适当修改。

![1.png](/1.png)
然后重启服务

```
systemctl restart zabbix-proxy.service
```

接下来就要添加主机了，也可以在原有主机上添加ipmi网络和用户

先添加ipmi网络接口，这里要确认zabbix\_proxy上能够访问到，端口默认623即可假如没有修改的话。

![2.png](/2.png)

添加用户，因为这里需要输入ipmi的用户和密码，所以这里建议新建一个无特殊的ipmi用户用于监控（由于我的超微服务器，认证算法直接默认即可，其他服务器请查看官方文档）

![3.png](/3.png)

完成后，还需要按需选择ipmi监控模板

![4.png](/4.png)

我这里选择的是"Template Server Chassis by IPMI"

没问题后直接点击更新，然后再次重启zabbix proxy服务

```
systemctl restart zabbix-proxy.service
```

我的博客即将同步至腾讯云开发者社区，邀请大家一同入驻：https://cloud.tencent.com/developer/support-plan?invite_code=pzrt9en9f15b
