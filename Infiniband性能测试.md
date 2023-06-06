---
title: Infiniband性能测试
description: 
published: true
date: 2023-06-06T15:02:00.875Z
tags: infiniband, rdma
editor: markdown
dateCreated: 2023-06-06T15:02:00.875Z
---



## ibping

先在一台服务器上使用ibping开启服务器模式，这台服务器的配置信息

```bash
[root@storage02 ~]# ibv_devices
    device                 node GUID
    ------              ----------------
    mlx4_0              0002c90300b382a0
    irdma0              ae1f6bfffeec331c
    irdma1              ae1f6bfffeec331d
[root@storage02 ~]# ibstat mlx4_0
CA 'mlx4_0'
        CA type: MT4099
        Number of ports: 2
        Firmware version: 2.42.5000
        Hardware version: 1
        Node GUID: 0x0002c90300b382a0
        System image GUID: 0x0002c90300b382a3
        Port 1:
                State: Down
                Physical state: Polling
                Rate: 10
                Base lid: 0
                LMC: 0
                SM lid: 0
                Capability mask: 0x02594868
                Port GUID: 0x0002c90300b382a1
                Link layer: InfiniBand
        Port 2:
                State: Active
                Physical state: LinkUp
                Rate: 56
                Base lid: 5
                LMC: 0
                SM lid: 1
                Capability mask: 0x0259486a
                Port GUID: 0x0002c90300b382a2
                Link layer: InfiniBand
```

```bash
[root@storage02 ~]# ibping -S -C mlx4_0 -P 2
```

然后另一台机器开启client模式

```bash
[root@storage03 ~]# ibv_devices
    device                 node GUID
    ------              ----------------
    mlx4_0              248a070300ea0390
    irdma0              ae1f6bfffeec32f2
    irdma1              ae1f6bfffeec32f3
[root@storage03 ~]# ibstat mlx4_0
CA 'mlx4_0'
        CA type: MT4099
        Number of ports: 1
        Firmware version: 2.42.5000
        Hardware version: 1
        Node GUID: 0x248a070300ea0390
        System image GUID: 0x248a070300ea0393
        Port 1:
                State: Active
                Physical state: LinkUp
                Rate: 56
                Base lid: 4
                LMC: 0
                SM lid: 1
                Capability mask: 0x0259486a
                Port GUID: 0x248a070300ea0391
                Link layer: InfiniBand
```

client端开始访问到server端

```bash
[root@storage03 ~]# ibping -G 0x0002c90300b382a2 -c 10
Pong from storage02.(none) (Lid 5): time 0.019 ms
Pong from storage02.(none) (Lid 5): time 0.021 ms
Pong from storage02.(none) (Lid 5): time 0.023 ms
Pong from storage02.(none) (Lid 5): time 0.017 ms
Pong from storage02.(none) (Lid 5): time 0.022 ms
Pong from storage02.(none) (Lid 5): time 0.022 ms
Pong from storage02.(none) (Lid 5): time 0.020 ms
Pong from storage02.(none) (Lid 5): time 0.019 ms
Pong from storage02.(none) (Lid 5): time 0.021 ms
Pong from storage02.(none) (Lid 5): time 0.021 ms

--- storage02.(none) (Lid 5) ibping statistics ---
10 packets transmitted, 10 received, 0% packet loss, time 10000 ms
rtt min/avg/max = 0.017/0.020/0.023 ms
```

## qperf

qperf跟ibping类似需要，但是qperf在测试时，建议关闭防火墙`systemctl stop firewalld`

在其中一台机器启用server模式

```bash
[root@storage01 ~]# qperf
```

另一台机器，查看频道适配器的配置

```bash 
[root@storage02 ~]# qperf -v -i mlx4_0:2 172.16.50.11 conf
conf:
    loc_node   =  storage02
    loc_cpu    =  20 Cores: Mixed CPUs
    loc_os     =  Linux 4.18.0-477.13.1.el8_8.x86_64
    loc_qperf  =  0.4.11
    rem_node   =  storage01
    rem_cpu    =  20 Cores: Mixed CPUs
    rem_os     =  Linux 4.18.0-477.13.1.el8_8.x86_64
    rem_qperf  =  0.4.11
```

也可以运行TCP带宽和延迟测试

```bash
[root@storage02 ~]# qperf 172.16.50.11 tcp_bw tcp_lat
tcp_bw:
    bw  =  3.96 GB/sec
tcp_lat:
    latency  =  11.1 us
```

也可以测试得到消息大小为1到64K的TCP延迟范围

```bash
[root@storage02 ~]# qperf 172.16.50.11 -oo msg_size:1:64K:*2 -vu tcp_lat
tcp_lat:
    latency   =  10.5 us
    msg_size  =     1 bytes
tcp_lat:
    latency   =  11.1 us
    msg_size  =     2 bytes
tcp_lat:
    latency   =  11.1 us
    msg_size  =     4 bytes
tcp_lat:
    latency   =  10.9 us
    msg_size  =     8 bytes
tcp_lat:
    latency   =  10.9 us
    msg_size  =    16 bytes
tcp_lat:
    latency   =  11.1 us
    msg_size  =    32 bytes
tcp_lat:
    latency   =  10.7 us
    msg_size  =    64 bytes
tcp_lat:
    latency   =  11.1 us
    msg_size  =   128 bytes
tcp_lat:
    latency   =  11.4 us
    msg_size  =   256 bytes
tcp_lat:
    latency   =  11.5 us
    msg_size  =   512 bytes
tcp_lat:
    latency   =  12.2 us
    msg_size  =     1 KiB (1,024)
tcp_lat:
    latency   =  13 us
    msg_size  =   2 KiB (2,048)
tcp_lat:
    latency   =  15.8 us
    msg_size  =     4 KiB (4,096)
tcp_lat:
    latency   =  17.2 us
    msg_size  =     8 KiB (8,192)
tcp_lat:
    latency   =  22.6 us
    msg_size  =    16 KiB (16,384)
tcp_lat:
    latency   =  38.8 us
    msg_size  =    32 KiB (32,768)
tcp_lat:
    latency   =  59.3 us
    msg_size  =    64 KiB (65,536)
```

## perftest

perftest提供了很全面的RDMA测试工具，具体工具可以参考[这里](https://github.com/linux-rdma/perftest)：

- 
  ib_send_lat 	latency test with send transactions

- ib_send_bw 	bandwidth test with send transactions
- ib_write_lat 	latency test with RDMA write transactions
- ib_write_bw 	bandwidth test with RDMA write transactions
- ib_read_lat 	latency test with RDMA read transactions
- ib_read_bw 	bandwidth test with RDMA read transactions
- ib_atomic_lat	latency test with atomic transactions
- ib_atomic_bw 	bandwidth test with atomic transactions

这里我以ib_read_lat作为示例，先在其中一台机器开启`ib_read_lat`

```bash
[root@storage03 ~]# ib_read_lat -d mlx4_0

************************************
* Waiting for client to connect... *
************************************
---------------------------------------------------------------------------------------
                    RDMA_Read Latency Test
 Dual-port       : OFF          Device         : mlx4_0
 Number of qps   : 1            Transport type : IB
 Connection type : RC           Using SRQ      : OFF
 PCIe relax order: ON
 ibv_wr* API     : OFF
 Mtu             : 2048[B]
 Link type       : IB
 Outstand reads  : 16
 rdma_cm QPs     : OFF
 Data ex. method : Ethernet
---------------------------------------------------------------------------------------
 local address: LID 0x04 QPN 0x020f PSN 0x17e736 OUT 0x10 RKey 0x8010100 VAddr 0x0055f4c06e7000
 remote address: LID 0x01 QPN 0x0215 PSN 0x79fcd6 OUT 0x10 RKey 0x28010100 VAddr 0x0055fa855bd000
---------------------------------------------------------------------------------------
```

然后另一台作为client去访问测试

```bash
[root@storage01 ~]# ib_read_lat -d mlx4_0 172.16.50.13
---------------------------------------------------------------------------------------
                    RDMA_Read Latency Test
 Dual-port       : OFF          Device         : mlx4_0
 Number of qps   : 1            Transport type : IB
 Connection type : RC           Using SRQ      : OFF
 PCIe relax order: ON
 ibv_wr* API     : OFF
 TX depth        : 1
 Mtu             : 2048[B]
 Link type       : IB
 Outstand reads  : 16
 rdma_cm QPs     : OFF
 Data ex. method : Ethernet
---------------------------------------------------------------------------------------
 local address: LID 0x01 QPN 0x0215 PSN 0x79fcd6 OUT 0x10 RKey 0x28010100 VAddr 0x0055fa855bd000
 remote address: LID 0x04 QPN 0x020f PSN 0x17e736 OUT 0x10 RKey 0x8010100 VAddr 0x0055f4c06e7000
---------------------------------------------------------------------------------------
 #bytes #iterations    t_min[usec]    t_max[usec]  t_typical[usec]    t_avg[usec]    t_stdev[usec]   99% percentile[usec]   99.9% percentile[usec]
Conflicting CPU frequency values detected: 2400.000000 != 1873.463000. CPU Frequency is not max.
 2       1000          1.76           6.12         1.79                1.80             0.06            1.88                    6.12
---------------------------------------------------------------------------------------
```

