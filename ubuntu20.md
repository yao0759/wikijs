---
title: ubuntu 20.04为mcx353a-fcbt安装驱动并配置IPoIB
description: 
published: true
date: 2022-09-03T07:21:08.750Z
tags: install, ubuntu, ipoib, infiniband
editor: markdown
dateCreated: 2022-09-03T07:21:08.750Z
---

# ubuntu 20.04为mcx353a-fcbt安装驱动并配置IPoIB

先在[Linux InfiniBand Drivers (nvidia.com)](https://network.nvidia.com/products/infiniband-drivers/linux/mlnx_ofed/)搜索适合的驱动，由于mellanox ConnectX-3 MCX353A-FCBT是比较久的版本，所以从MLNX_OFED 5.1开始，就不开始支持下属版本

- ConnectX-3 Pro
- ConnectX-3
- Connect-IB
- RDMA experimental verbs library (mlnx_lib)

所以我们选择的版本是4.9.0-5.1.0版本

开始下载

```shell
wget https://content.mellanox.com/ofed/MLNX_OFED-4.9-5.1.0.0/MLNX_OFED_LINUX-4.9-5.1.0.0-ubuntu20.04-x86_64.tgz
```

解压文件

```shell
tar zxf MLNX_OFED_LINUX-4.9-5.1.0.0-ubuntu20.04-x86_64.tgz
```

开始安装

```shell
cd MLNX_OFED_LINUX-4.9-5.1.0.0-ubuntu20.04-x86_64
```

```shell
./mlnxofedinstall
```

开始安装后，会提示需要安装依赖包，输入y确认即可，然后等待编译安装完成

```shell
......
This program will install the MLNX_OFED_LINUX package on your machine.
Note that all other Mellanox, OEM, OFED, RDMA or Distribution IB packages will be removed.
Those packages are removed due to conflicts with MLNX_OFED_LINUX, do not reinstall them.

Do you want to continue?[y/N]:y
```

完成后会提示下述信息

```shell
......
Real log file: /tmp/MLNX_OFED_LINUX.2274.logs/fw_update.log
Device (65:00.0):
        65:00.0 Network controller: Mellanox Technologies MT27500 Family [ConnectX-3]
        Link Width: x8
        PCI Link Speed: 8GT/s

Installation passed successfully
To load the new driver, run:
/etc/init.d/openibd restart
```

按照它的提示，重启服务，重新加载驱动

```shell
/etc/init.d/openibd restart
```

使用ibstat命令查看设备信息，有设备信息则代表驱动有被正常安装

```shell
# ibstat
CA 'mlx4_0'
        CA type: MT4099
        Number of ports: 1
        Firmware version: 2.42.5000
        Hardware version: 1
        Node GUID: 0x0002c903004198d0
        System image GUID: 0x0002c903004198d3
        Port 1:
                State: Down
                Physical state: Polling
                Rate: 10
                Base lid: 0
                LMC: 0
                SM lid: 0
                Capability mask: 0x02514868
                Port GUID: 0x0002c903004198d1
                Link layer: InfiniBand
```

开始配置netplan，ubuntu 20.04 默认使用netplan配置网络

```shell
# vim /etc/netplan/00-installer-config.yaml

# This is the network config written by 'subiquity'
network:
  ethernets:
    eno1:
      dhcp4: true
    eno2:
      dhcp4: true
    enp183s0f0:
      dhcp4: true
    enp183s0f1:
      dhcp4: true
    ibp182s0:
      addresses:
      - 172.16.50.2/16
  version: 2
```

配置完成后，需要开启opensmd服务

```
/etc/init.d/opensmd start 
```

