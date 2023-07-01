---
title: virtiofs性能
description: 
published: true
date: 2023-07-01T14:49:35.397Z
tags: kvm, virtiofs
editor: markdown
dateCreated: 2023-07-01T14:39:34.778Z
---

# virtiofs性能

## 什么是virtiofs

virtiofs是红帽在kata社区提出的一个共享文件系统的解决方案。社区地址：[virtio-fs - shared file system for virtual machines](https://virtio-fs.gitlab.io/)

## 为什么不用9p？

这个问题的现有解决方案，如virtio-9p，是基于现有的网络协议，没有针对虚拟化用例进行优化。因此，它们的性能不如本地文件系统，也不能提供一些应用程序所依赖的语义。

Virtio-fs利用了虚拟机与管理程序共处的优势，避免了与网络文件系统相关的开销。

## 来看看virtiofs的性能怎么样？

这里测试的host系统版本为rocky 9，guest系统版本为ubuntu 20.04。测试工具选择fio，测试的内容为随机读写。共享的路径为一块3TB的HDD。

host写入性能测试结果：

```bash
[root@nas shared]# fio --name=random-write --ioengine=posixaio --rw=randwrite --bs=4k --numjobs=1 --size=4g --iodepth=1 --runtime=60 --time_based --end_fsync=1
random-write: (g=0): rw=randwrite, bs=(R) 4096B-4096B, (W) 4096B-4096B, (T) 4096B-4096B, ioengine=posixaio, iodepth=1
fio-3.27
Starting 1 process
random-write: Laying out IO file (1 file / 4096MiB)
Jobs: 1 (f=1): [w(1)][100.0%][eta 00m:00s]
random-write: (groupid=0, jobs=1): err= 0: pid=32746: Sat Jul  1 22:11:49 2023
  write: IOPS=27.9k, BW=109MiB/s (114MB/s)(8192MiB/75042msec); 0 zone resets
    slat (nsec): min=480, max=341422, avg=777.19, stdev=358.95
    clat (nsec): min=250, max=2553.1k, avg=6280.18, stdev=2220.15
     lat (usec): min=5, max=2554, avg= 7.06, stdev= 2.26
    clat percentiles (nsec):
     |  1.00th=[ 5280],  5.00th=[ 5472], 10.00th=[ 5600], 20.00th=[ 5728],
     | 30.00th=[ 5984], 40.00th=[ 6112], 50.00th=[ 6176], 60.00th=[ 6240],
     | 70.00th=[ 6368], 80.00th=[ 6496], 90.00th=[ 6752], 95.00th=[ 7328],
     | 99.00th=[ 9792], 99.50th=[10944], 99.90th=[20864], 99.95th=[23680],
     | 99.99th=[35584]
   bw (  KiB/s): min=73064, max=564064, per=100.00%, avg=508400.24, stdev=80466.79, samples=33
   iops        : min=18266, max=141016, avg=127100.12, stdev=20116.72, samples=33
  lat (nsec)   : 500=0.01%
  lat (usec)   : 4=0.01%, 10=99.14%, 20=0.75%, 50=0.10%, 100=0.01%
  lat (usec)   : 250=0.01%, 500=0.01%
  lat (msec)   : 4=0.01%
  cpu          : usr=4.34%, sys=7.33%, ctx=2241649, majf=0, minf=20
  IO depths    : 1=100.0%, 2=0.0%, 4=0.0%, 8=0.0%, 16=0.0%, 32=0.0%, >=64=0.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     issued rwts: total=0,2097153,0,0 short=0,0,0,0 dropped=0,0,0,0
     latency   : target=0, window=0, percentile=100.00%, depth=1

Run status group 0 (all jobs):
  WRITE: bw=109MiB/s (114MB/s), 109MiB/s-109MiB/s (114MB/s-114MB/s), io=8192MiB (8590MB), run=75042-75042msec

Disk stats (read/write):
  sdc: ios=0/52057, merge=0/15059, ticks=0/271447, in_queue=271681, util=79.50%
```

host读取性能测试结果：

```bash
[root@nas shared]# fio --name=random-read --ioengine=posixaio --rw=randread --bs=4k --numjobs=1 --size=4g --iodepth=1 --runtime=60 --time_based --end_fsync=1
random-read: (g=0): rw=randread, bs=(R) 4096B-4096B, (W) 4096B-4096B, (T) 4096B-4096B, ioengine=posixaio, iodepth=1
fio-3.27
Starting 1 process
random-read: Laying out IO file (1 file / 4096MiB)
Jobs: 1 (f=1): [r(1)][100.0%][r=648KiB/s][r=162 IOPS][eta 00m:00s]
random-read: (groupid=0, jobs=1): err= 0: pid=32779: Sat Jul  1 22:13:51 2023
  read: IOPS=165, BW=661KiB/s (677kB/s)(38.7MiB/60003msec)
    slat (nsec): min=942, max=170530, avg=3941.71, stdev=2372.65
    clat (usec): min=59, max=42472, avg=6040.74, stdev=2574.32
     lat (usec): min=61, max=42474, avg=6044.68, stdev=2574.39
    clat percentiles (usec):
     |  1.00th=[  161],  5.00th=[ 2089], 10.00th=[ 2638], 20.00th=[ 3523],
     | 30.00th=[ 4359], 40.00th=[ 5211], 50.00th=[ 6063], 60.00th=[ 6849],
     | 70.00th=[ 7767], 80.00th=[ 8586], 90.00th=[ 9372], 95.00th=[ 9896],
     | 99.00th=[10683], 99.50th=[10945], 99.90th=[11338], 99.95th=[15533],
     | 99.99th=[42730]
   bw (  KiB/s): min=  592, max=  744, per=99.96%, avg=661.38, stdev=32.87, samples=119
   iops        : min=  148, max=  186, avg=165.34, stdev= 8.22, samples=119
  lat (usec)   : 100=0.35%, 250=0.83%, 1000=0.02%
  lat (msec)   : 2=3.01%, 4=21.54%, 10=69.79%, 20=4.45%, 50=0.01%
  cpu          : usr=0.22%, sys=0.08%, ctx=9934, majf=0, minf=16
  IO depths    : 1=100.0%, 2=0.0%, 4=0.0%, 8=0.0%, 16=0.0%, 32=0.0%, >=64=0.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     issued rwts: total=9919,0,0,0 short=0,0,0,0 dropped=0,0,0,0
     latency   : target=0, window=0, percentile=100.00%, depth=1

Run status group 0 (all jobs):
   READ: bw=661KiB/s (677kB/s), 661KiB/s-661KiB/s (677kB/s-677kB/s), io=38.7MiB (40.6MB), run=60003-60003msec

Disk stats (read/write):
  sdc: ios=9917/6, merge=0/4, ticks=59284/51, in_queue=59372, util=99.85%
```

guest写入性能测试结果：

```bash
root@guest:/mnt# fio --name=random-write --ioengine=posixaio --rw=randwrite --bs=4k --numjobs=1 --size=4g --iodepth=1 --runtime=60 --time_based --end_fsync=1
random-write: (g=0): rw=randwrite, bs=(R) 4096B-4096B, (W) 4096B-4096B, (T) 4096B-4096B, ioengine=posixaio, iodepth=1
fio-3.16
Starting 1 process
Jobs: 1 (f=1): [F(1)][100.0%][eta 00m:00s]
random-write: (groupid=0, jobs=1): err= 0: pid=1641: Sat Jul  1 14:21:33 2023
  write: IOPS=8445, BW=32.0MiB/s (34.6MB/s)(3397MiB/102979msec); 0 zone resets
    slat (nsec): min=491, max=777322, avg=3441.18, stdev=2428.50
    clat (nsec): min=851, max=26784k, avg=64188.84, stdev=32389.57
     lat (usec): min=41, max=26789, avg=67.63, stdev=32.49
    clat percentiles (usec):
     |  1.00th=[   51],  5.00th=[   55], 10.00th=[   56], 20.00th=[   58],
     | 30.00th=[   60], 40.00th=[   61], 50.00th=[   62], 60.00th=[   64],
     | 70.00th=[   66], 80.00th=[   70], 90.00th=[   75], 95.00th=[   80],
     | 99.00th=[  110], 99.50th=[  121], 99.90th=[  163], 99.95th=[  227],
     | 99.99th=[  375]
   bw (  KiB/s): min=46664, max=63920, per=100.00%, avg=57988.20, stdev=3338.82, samples=119
   iops        : min=11666, max=15980, avg=14497.03, stdev=834.72, samples=119
  lat (nsec)   : 1000=0.01%
  lat (usec)   : 2=0.01%, 4=0.01%, 10=0.01%, 20=0.01%, 50=0.82%
  lat (usec)   : 100=97.52%, 250=1.62%, 500=0.03%, 750=0.01%, 1000=0.01%
  lat (msec)   : 2=0.01%, 10=0.01%, 50=0.01%
  cpu          : usr=6.01%, sys=4.72%, ctx=871421, majf=0, minf=49
  IO depths    : 1=100.0%, 2=0.0%, 4=0.0%, 8=0.0%, 16=0.0%, 32=0.0%, >=64=0.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     issued rwts: total=0,869750,0,1 short=0,0,0,0 dropped=0,0,0,0
     latency   : target=0, window=0, percentile=100.00%, depth=1

Run status group 0 (all jobs):
  WRITE: bw=32.0MiB/s (34.6MB/s), 32.0MiB/s-32.0MiB/s (34.6MB/s-34.6MB/s), io=3397MiB (3562MB), run=102979-102979msec

```

guest读取性能测试结果，这里的结果很有趣啊，可能是因为缓存的原因，随机读取的结果很好，即便我执行`echo 3 > /proc/sys/vm/drop_caches`后测试也一样，有可能随机读取创建的文件并未真实写入到真实的文件系统下

```bash
[root@guest:/mnt# fio --name=random-read --ioengine=posixaio --rw=randread --bs=4k --numjobs=1 --size=4g --iodepth=1 --runtime=60 --time_based --end_fsync=1
random-read: (g=0): rw=randread, bs=(R) 4096B-4096B, (W) 4096B-4096B, (T) 4096B-4096B, ioengine=posixaio, iodepth=1
fio-3.16
Starting 1 process
random-read: Laying out IO file (1 file / 4096MiB)
Jobs: 1 (f=1): [r(1)][100.0%][r=76.8MiB/s][r=19.7k IOPS][eta 00m:00s]
random-read: (groupid=0, jobs=1): err= 0: pid=1696: Sat Jul  1 14:25:27 2023
  read: IOPS=19.5k, BW=76.3MiB/s (79.0MB/s)(4578MiB/60001msec)
    slat (nsec): min=441, max=1973.3k, avg=3260.84, stdev=4183.31
    clat (nsec): min=591, max=4888.2k, avg=46547.74, stdev=14387.57
     lat (usec): min=28, max=4894, avg=49.81, stdev=15.13
    clat percentiles (usec):
     |  1.00th=[   37],  5.00th=[   40], 10.00th=[   42], 20.00th=[   43],
     | 30.00th=[   44], 40.00th=[   45], 50.00th=[   47], 60.00th=[   47],
     | 70.00th=[   48], 80.00th=[   50], 90.00th=[   52], 95.00th=[   55],
     | 99.00th=[   62], 99.50th=[   72], 99.90th=[  165], 99.95th=[  227],
     | 99.99th=[  494]
   bw (  KiB/s): min=33424, max=86672, per=99.99%, avg=78116.44, stdev=6079.56, samples=119
   iops        : min= 8356, max=21668, avg=19529.08, stdev=1519.90, samples=119
  lat (nsec)   : 750=0.01%, 1000=0.01%
  lat (usec)   : 2=0.01%, 4=0.01%, 50=82.42%, 100=17.36%, 250=0.19%
  lat (usec)   : 500=0.03%, 750=0.01%, 1000=0.01%
  lat (msec)   : 2=0.01%, 4=0.01%, 10=0.01%
  cpu          : usr=4.10%, sys=19.49%, ctx=1172447, majf=0, minf=50
  IO depths    : 1=100.0%, 2=0.0%, 4=0.0%, 8=0.0%, 16=0.0%, 32=0.0%, >=64=0.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     issued rwts: total=1171868,0,0,0 short=0,0,0,0 dropped=0,0,0,0
     latency   : target=0, window=0, percentile=100.00%, depth=1

Run status group 0 (all jobs):
   READ: bw=76.3MiB/s (79.0MB/s), 76.3MiB/s-76.3MiB/s (79.0MB/s-79.0MB/s), io=4578MiB (4800MB), run=60001-60001msec](<root@cvmfs:/mnt# fio --name=random-read --ioengine=posixaio --rw=randread --bs=4k --numjobs=1 --size=4g --iodepth=1 --runtime=60 --time_based --end_fsync=1
random-read: (g=0): rw=randread, bs=(R) 4096B-4096B, (W) 4096B-4096B, (T) 4096B-4096B, ioengine=posixaio, iodepth=1
fio-3.16
Starting 1 process
Jobs: 1 (f=1): [r(1)][100.0%][r=94.8MiB/s][r=24.3k IOPS][eta 00m:00s]
random-read: (groupid=0, jobs=1): err= 0: pid=1730: Sat Jul  1 14:29:51 2023
  read: IOPS=19.7k, BW=76.8MiB/s (80.5MB/s)(4606MiB/60001msec)
    slat (nsec): min=472, max=1074.7k, avg=3219.88, stdev=2672.15
    clat (nsec): min=922, max=7624.5k, avg=46275.44, stdev=17124.54
     lat (usec): min=24, max=7630, avg=49.50, stdev=17.60
    clat percentiles (usec):
     |  1.00th=[   31],  5.00th=[   35], 10.00th=[   39], 20.00th=[   42],
     | 30.00th=[   44], 40.00th=[   45], 50.00th=[   46], 60.00th=[   48],
     | 70.00th=[   49], 80.00th=[   51], 90.00th=[   53], 95.00th=[   56],
     | 99.00th=[   65], 99.50th=[   80], 99.90th=[  153], 99.95th=[  217],
     | 99.99th=[  392]
   bw (  KiB/s): min=48880, max=106680, per=99.78%, avg=78431.01, stdev=8628.48, samples=119
   iops        : min=12220, max=26670, avg=19607.72, stdev=2157.12, samples=119
  lat (nsec)   : 1000=0.01%
  lat (usec)   : 2=0.01%, 4=0.01%, 50=78.75%, 100=20.98%, 250=0.24%
  lat (usec)   : 500=0.03%, 750=0.01%, 1000=0.01%
  lat (msec)   : 2=0.01%, 4=0.01%, 10=0.01%
  cpu          : usr=1.67%, sys=19.68%, ctx=1179722, majf=0, minf=47
  IO depths    : 1=100.0%, 2=0.0%, 4=0.0%, 8=0.0%, 16=0.0%, 32=0.0%, %3E=64=0.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     issued rwts: total=1179112,0,0,0 short=0,0,0,0 dropped=0,0,0,0
     latency   : target=0, window=0, percentile=100.00%, depth=1

Run status group 0 (all jobs):
   READ: bw=76.8MiB/s (80.5MB/s), 76.8MiB/s-76.8MiB/s (80.5MB/s-80.5MB/s), io=4606MiB (4830MB), run=60001-60001msec>)
```

重新测试，现在guest下跑测试过程，并同步在guest和host校验创建的测试文件的md5sum，发现md5值校验一致。看来文件已经完全写入到文件系统下了。

```
[root@nas shared]# md5sum random-read.0.0
a8d05ca10efe386534cd457bf6ccb85c  random-read.0.0
root@cvmfs:~# md5sum /mnt/random-read.0.0
a8d05ca10efe386534cd457bf6ccb85c  /mnt/random-read.0.0
```

我们来看看virtiofs的配置是怎样的

```
<filesystem type='mount' accessmode='passthrough'>
      <driver type='virtiofs'/>
      <source dir='/shared/'/>
      <target dir='hostshare'/>
      <address type='pci' domain='0x0000' bus='0x07' slot='0x00' function='0x0'/>
    </filesystem>
```

为什么有这样的性能呢，从[libvirt: Sharing files with Virtiofs](https://libvirt.org/kbase/virtiofs.html)可能提供了答案，这个链接中有这么一个说明

> Almost all virtio devices (all that use virtqueues) require access to at least certain portions of guest RAM (possibly policed by DMA). In case of virtiofsd, much like in case of other vhost-user (see https://www.qemu.org/docs/master/interop/vhost-user.html) virtio devices that are realized by an userspace process, this in practice means that QEMU needs to allocate the backing memory for all the guest RAM as shared memory. As of QEMU 4.2, it is possible to explicitly specify a memory backend when specifying the NUMA topology. This method is however only viable for machine types that do support NUMA. As of QEMU 5.0.0 and libvirt 6.9.0, it is possible to specify the memory backend without NUMA (using the so called memobject interface).

也就是说几乎所有的virtio设备（所有使用virtqueues的设备）都需要访问至少某些部分的客户RAM（可能由DMA控制）。对于virtiofsd来说，就像其他由用户空间进程实现的vhost-user virtio设备一样，这实际上意味着QEMU需要为所有的客户RAM分配支持内存作为共享内存。

所以读取文件时，其实访问的可能是内存，这样的测试结果也只可能是内存速度。