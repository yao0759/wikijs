---
title: rocky 8 如何配置infiniband网络
description: 
published: true
date: 2023-06-06T15:03:12.860Z
tags: infiniband, rdma, rocky
editor: markdown
dateCreated: 2023-06-06T15:03:12.860Z
---



这里使用IB卡型号是mellanox ConnectX-3 MCX353A-FCBT。rocky自带了IB驱动，直接能够识别IB网口，但要在NetworkManager配置生效还需要做一些操作。

建议安装Infiniband Support包。

```bash
dnf groupinstall -y "Infiniband"
```

然后查看端口配型，需要安装mstflint工具

```
dnf install mstflint
```

检查ib的设备的pci地址

```bash
[root@storage01 ~]# lspci | grep Mellanox -i
65:00.0 Network controller: Mellanox Technologies MT27500 Family [ConnectX-3]
```

然后修改网口类型，从下述可以看到它原本的模式是VPI，这个类型无法配置IPoIB，我们将它修改为IB类型，然后重启机器

```bash
[root@storage01 ~]# mstconfig -d 65:00.0 s LINK_TYPE_P1=IB

Device #1:
----------

Device type:    ConnectX3
Device:         65:00.0

Configurations:                                      Next Boot       New
         LINK_TYPE_P1                                VPI(3)          IB(1)

 Apply new Configuration? (y/n) [n] : y
Applying... Done!
-I- Please reboot machine to load new configurations.
[root@storage01 ~]#
```

重启完成后，我们可以用nmtui添加infiniband配置了

![image-20230606110900603](https://yhblog-1254039996.cos.ap-guangzhou.myqcloud.com/img-blog/image-20230606110900603.png)

![image-20230606110945015](https://yhblog-1254039996.cos.ap-guangzhou.myqcloud.com/img-blog/image-20230606110945015.png)

可以修改成下述配置（注意：IPoIB 设备可在 `Datagram` 或 `Connected` 模式中配置。两者的区别可以参考这里[第 6 章 配置 IPoIB Red Hat Enterprise Linux 8 | Red Hat Customer Portal](https://access.redhat.com/documentation/zh-cn/red_hat_enterprise_linux/8/html/configuring_infiniband_and_rdma_networks/configuring-ipoib_configuring-infiniband-and-rdma-networks)）

![image-20230606111052701](https://yhblog-1254039996.cos.ap-guangzhou.myqcloud.com/img-blog/image-20230606111052701.png)