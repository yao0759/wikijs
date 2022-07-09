---
title: ovirt配置虚拟机如何使用cloud-init
description: 
published: true
date: 2022-07-09T08:12:47.118Z
tags: kvm, ovirt, cloud-init
editor: markdown
dateCreated: 2022-07-02T15:23:04.261Z
---

# ovirt虚拟机如何使用cloud-init

最近在使用ovirt时，自己创建的template在配置cloud-init一些选项并没有生效，去踩了一些坑，整理了一些常见问题

1. cloud-init配置只在机器首次启动时才能生效

2. cloud-init在配置网络时的网络协议建议选用openstack元数据

3. **基于虚拟机创建的template，建议在创建前将网络配置文件删除，不然cloud-init的网络配置无法生效**。

4. 要确认虚拟机中的cloud-init服务开机启动

   ```shell
   systemctl enable cloud-init-local.service
   systemctl start cloud-init-local.service
   systemctl enable cloud-init.service
   systemctl start cloud-init.service
   systemctl enable cloud-config.service
   systemctl start cloud-config.service
   systemctl enable cloud-final.service
   systemctl start cloud-final.service
   ```

   