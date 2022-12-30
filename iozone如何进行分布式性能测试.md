---
title: iozone如何进行分布式性能测试
description: 
published: true
date: 2022-12-30T15:35:31.314Z
tags: iozone
editor: markdown
dateCreated: 2022-12-30T15:35:31.314Z
---

我们去查看iozone的man手册可以看到，这一段

```
-+m filename
              Used to specify a filename that will be used to specify the clients in a distributed measurement. The  file contains  one  line for each client. The fields are space delimited. Field 1 is the client name. Field 2 is the working directory, on the client, where Iozone will run. Field 3 is the path to the  executable  Iozone on the client.
```

由上述信息可知，配置文件有三行信息，第一行是机器名或者IP；第二行是挂载目录；第三行则是iozone的执行路径

```
[Clinet]    [mount_directory]   /usr/bin/iozone
```

而且在测试开始前，客户端之间要配置免密登录，配置完免密登录后，我们可以创建一个配置文件host.cfg

```
storage01  /mnt/  /opt/iozone
storage02  /mnt/  /opt/iozone
storage03  /mnt/  /opt/iozone
storage04  /mnt/  /opt/iozone
```

配置完，就可以开始测试了

```
export RSH=ssh
/opt/iozone -ceIT -i0 -i1 -i2 -+n -r 4k -s1g  -+m host.cfg -t4 -Rb /root/4k_distributed_performance_test.xls
```

执行完应该会输出下述信息

```
root@storage01:~# /opt/iozone -ceIT -i0 -i1 -i2 -+n -r 4k -s1g  -+m host.cfh -t4 -Rb /root/4k_distributed_performance_test.xls
        Iozone: Performance Test of File I/O
                Version $Revision: 3.492 $
                Compiled for 64 bit mode.
                Build: linux-AMD64

        Contributors:William Norcott, Don Capps, Isom Crawford, Kirby Collins
                     Al Slater, Scott Rhine, Mike Wisner, Ken Goss
                     Steve Landherr, Brad Smith, Mark Kelly, Dr. Alain CYR,
                     Randy Dunlap, Mark Montague, Dan Million, Gavin Brebner,
                     Jean-Marc Zucconi, Jeff Blomberg, Benny Halevy, Dave Boone,
                     Erik Habbinga, Kris Strecker, Walter Wong, Joshua Root,
                     Fabrice Bacchella, Zhenghua Xue, Qin Li, Darren Sawyer,
                     Vangel Bojaxhi, Ben England, Vikentsi Lapa,
                     Alexey Skidanov, Sudhir Kumar.

        Run began: Fri Oct 21 11:39:44 2022

        Include close in write timing
        Include fsync in write timing
        O_DIRECT feature enabled
        No retest option selected
        Record Size 4 kB
        File size set to 1048576 kB
        Network distribution mode enabled.
        Excel chart generation enabled
        Command line used: /opt/iozone -ceIT -i0 -i1 -i2 -+n -r 4k -s1g -+m host.cfh -t4 -Rb /root/4k_distributed_performance_test.xls
        Output is in kBytes/sec
        Time Resolution = 0.000001 seconds.
        Processor cache size set to 1024 kBytes.
        Processor cache line size set to 32 bytes.
        File stride size set to 17 * record size.
        Throughput test with 4 threads
        Each thread writes a 1048576 kByte file in 4 kByte records
```