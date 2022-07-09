---
title: zabbix关于Utilization of trapper processes over报警的处理及优化
description: 
published: true
date: 2022-07-09T08:13:20.798Z
tags: zabbix
editor: markdown
dateCreated: 2022-06-23T10:05:41.970Z
---

**简介：** zabbix关于Utilization of trapper processes over报警的处理及优化

![image-20220322160421190.png](https://ucc.alicdn.com/pic/developer-ecology/2dee7fa43e6748e4803ecb9339aaba25.png "image-20220322160421190.png")

在将StartPollers值设置为20后，虽然还是会触发报警但是在保持5min后数值还是能够下降

![image-20220323100327517.png](https://ucc.alicdn.com/pic/developer-ecology/3f3703d7855345c1b42f88edd29fd7fa.png "image-20220323100327517.png")

将StartPollers值设置为40继续监控，要注意假如zabbix server内存过小，StartPollers过大会占用太多的内存资源。

## 参考链接

[zabbix poller process more than 75% busy - ZABBIX Forums\](https://www.zabbix.com/forum/zabbix-troubleshooting-and-problems/400273-zabbix-poller-process-more-than-75-busy)

