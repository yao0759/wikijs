---
title: zabbix 5.0如何将esxi6.7添加到监控
description: 
published: true
date: 2022-07-09T08:13:13.595Z
tags: zabbix, esxi
editor: markdown
dateCreated: 2022-06-23T12:32:14.315Z
---

# zabbix 5.0如何将esxi6.7添加到监控

今天有个需求，需要将一台esxi 6.7 server添加到我们的zabbix监控服务器上，将我做的操作踩的一点坑写出来

## 配置

在配置前，我们需要先修改zabbix server配置文件/etc/zabbix/zabbix\_server.conf，启用StartVMwareCollectors参数，默认是不启用的。这里的值我仅设置为2，因为我仅仅需要监控一台esxi服务器，假如有更多的监控数量，可以参考这个servicenum < StartVMwareCollectors < (servicenum \* 2)去设置。

![image-20220512221306521.png](https://ucc.alicdn.com/pic/developer-ecology/41c9972a0d514c9bbd9948c4aa15b04b.png "image-20220512221306521.png")

修改成功后，重启zabbix-server服务

systemctl restart zabbix-server

然后我们需要在esxi中开启调试功能，在esxi主机管理的高级设置中可以找到Config.HostAgent.plugins.solo.enableMob，将默认的false修改为true，如下图所示

![image-20220512140438167.png](https://ucc.alicdn.com/pic/developer-ecology/b3682c881b4e4266b4278a1feb7b4b84.png "image-20220512140438167.png")

保存后，还需要添加一个独立的只读用户去让zabbix获取esxi参数

![image-20220512221759335.png](https://ucc.alicdn.com/pic/developer-ecology/ac1473b7375043c69431a2ae5caf5d9e.png "image-20220512221759335.png")

添加后赋予权限(主机-->操作-->权限)

![image-20220512221902965.png](https://ucc.alicdn.com/pic/developer-ecology/8fcab42b0991429ca663384cc7280ba1.png "image-20220512221902965.png")

![image-20220512140803969.png](https://ucc.alicdn.com/pic/developer-ecology/a79c25da41124e00950e494652c8c9ad.png "image-20220512140803969.png")

配置完成后，我们需要打开https://<ip>/mob/?moid=ha-host&doPath=hardware.systemInfo查看主机的uuid

![image-20220512222308732.png](https://ucc.alicdn.com/pic/developer-ecology/cb293455d0c6432192851c7422554657.png "image-20220512222308732.png")

完成后，我们就可以开始在zabbix中添加主机了，这里的主机名称必须是UUID

![image-20220512222431289.png](https://ucc.alicdn.com/pic/developer-ecology/401c4419ab834add97f538e7ccd2ad50.png "image-20220512222431289.png")

![image-20220512222714112.png](https://ucc.alicdn.com/pic/developer-ecology/90683053f9104353a43498dbba281e1d.png "image-20220512222714112.png")

然后添加宏，这里有一个问题需要注意，因为5.0的变更，宏值现在是{$VMWARE.URL}

、{$VMWARE.USERNAME}、{$VMWARE.PASSWORD}、{$VMWARE.HV.UUID}

![image-20220512222636231.png](https://ucc.alicdn.com/pic/developer-ecology/47f1564e0e8c47529e26e810cc9b00d9.png "image-20220512222636231.png")

-   {$VMWARE.URL}：这里填https://<esxi\_ip>/sdk
-   {$VMWARE.USERNAME}：创建的esxi用户
-   {$VMWARE.PASSWORD}：创建用户的密码
-   {$VMWARE.HV.UUID}：esxi的uuid