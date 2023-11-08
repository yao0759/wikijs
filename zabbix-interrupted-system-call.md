---
title: zabbix-agent问题提示interrupted system call
description: 
published: true
date: 2023-11-08T14:19:19.061Z
tags: zabbix
editor: markdown
dateCreated: 2023-11-08T14:19:19.061Z
---

agent日志

```
 36779:20231020:100243.379 **** Enabled features ****
 36779:20231020:100243.379 IPv6 support:          YES
 36779:20231020:100243.379 TLS support:           YES
 36779:20231020:100243.379 **************************
 36779:20231020:100243.379 using configuration file: /etc/zabbix/zabbix_agentd.conf
 36779:20231020:100243.379 agent #0 started [main process]
 36780:20231020:100243.380 agent #1 started [collector]
 36781:20231020:100243.381 agent #2 started [listener #1]
 36782:20231020:100243.382 agent #3 started [listener #2]
 36783:20231020:100243.383 agent #4 started [listener #3]
 36783:20231020:100726.159 Message from 192.168.1.178 is missing header. Message ignored.
```

server dashboard信息

![image-20231020100922250](https://yhblog-1254039996.cos.ap-guangzhou.myqcloud.com/img-blog/image-20231020100922250.png)



这种情况一般是安装的agent不完整需要重新安装，重新安装后就正常了