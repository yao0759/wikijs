---
title: 服务器上的RTC时间与世界时间不一致解决办法
description: 
published: true
date: 2022-07-09T08:13:31.805Z
tags: rtc
editor: markdown
dateCreated: 2022-06-23T12:26:56.871Z
---

**简介：** 服务器上的RTC时间与世界时间不一致解决办法

最近发现服务器的时钟时间跟世界时间不一致、

无论怎么修改ntp server都不行，data命令查看比世界时间快了20分钟左右，使用timedatctl命令查看，发现显示的是RTC时间

```
[root@storage2 ~]# date
Wed May 11 10:11:02 UTC 2022
[root@storage2 ~]# timedatectl status
      Local time: Wed 2022-05-11 09:50:40 UTC
  Universal time: Wed 2022-05-11 09:50:40 UTC
        RTC time: Wed 2022-05-11 10:11:20
       Time zone: UTC (UTC, +0000)
     NTP enabled: yes
NTP synchronized: no
 RTC in local TZ: no
      DST active: n/a
```

需要hwclock重新同步才行
```
sudo hwclock --systohc
```

重新同步后查看

```
[root@storage2 ~] date
Wed May 11 09:53:43 UTC 2022
[root@storage2 ~] timedatectl status
      Local time: Wed 2022-05-11 09:53:50 UTC
  Universal time: Wed 2022-05-11 09:53:50 UTC
        RTC time: Wed 2022-05-11 09:53:50
       Time zone: UTC (UTC, +0000)
     NTP enabled: yes
NTP synchronized: no
 RTC in local TZ: no
      DST active: n/a
```
