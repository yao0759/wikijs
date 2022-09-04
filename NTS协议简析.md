---
title: NTS协议简析
description: 
published: true
date: 2022-09-04T04:19:19.857Z
tags: nts, ntp
editor: markdown
dateCreated: 2022-09-04T04:19:19.857Z
---

目前互联网中ntp是仍在使用不安全的互联网协议之一。后来Cloudflare发布了NTS协议，去保证NTP的安全。

## NTP传统请求过程

1.  客户端向NTP服务器发送查询包
2.  服务器用时钟时间进行响应
3.  客户端计算其时钟和远程时钟之间的差值得到估计值，并试图补偿其中的网络延迟

注意：NTP 客户端会查询多个服务器并实施算法来选择最佳估计值，并拒绝明显错误的答案。

## NTS连接过程

-   协商在第二阶段使用的[AEAD](https://en.wikipedia.org/wiki/Authenticated_encryption)算法；
-   协商第二个协议（目前，标准只定义了 NTS 如何与 NTPv4 协作）；
-   协商 NTP 服务器的 IP 地址和端口；
-   创建第二阶段使用的 cookie；
-   从 TLS 会话创建两个对称秘钥（C2S 和 S2C）。

## 支持NTS协议的服务器

目前支持NTS的公共服务器非常少，主要的提供商有Cloudflare和Netnod

-   [time.cloudflare.com](http://time.cloudflare.com/)
-   [sth1.nts.netnod.se:4460](http://sth1.nts.netnod.se:4460/)

## 如何使用NTS

ntpclient现在有两个版本，一个是python版本，一个是go版本，另外则是chrony已经有分支支持nts。fedora从33版本已经开始支持NTS了。

### 条件

系统仅支持**Debian 9 (Stretch), Debian 10 (Buster), Ubuntu 16.04 LTS (Xenial Xerus) , Ubuntu 18.04 LTS (Bionic Beaver).**

### 编译ntsclient

拉取库

```shell
git clone https://gitlab.com/hacklunch/ntsclient; cd ntsclient; make
```

假如不想ntsclient以root用户运行

```shell
sudo setcap CAP_SYS_TIME+ep ./ntsclient
```

执行

```shell
./ntsclient --config ntsclient.toml
```

### 编译NTPsec

拉取库，使用./buildprep安装构建需要的包

```shell
git clone https://gitlab.com/NTPsec/ntpsec.git; cd ntpsec;sudo ./buildprep
```

构建完成后，可以用waf构建NTPsec

```shell
./waf configure
./waf build
```

设置配置文件，创建ntp.conf

```shell
# Exchange time with everybody, but don't allow configuration.
# This is the right security setup for 99% of deployments.
restrict default kod limited nomodify nopeer noquery
restrict -6 default kod limited nomodify nopeer noquery

# Local users may interrogate the NTP server more closely.
restrict 127.0.0.1
restrict -6 ::1

# Minimal logging - we declare a drift file and that's it.
driftfile /var/lib/ntp/ntp.drift

 server nts.netnod.se:4460 nts iburst
 server sth1.nts.netnod.se:4460 nts iburst
 server sth2.nts.netnod.se:4460 nts iburst
```

开始测试，测试前需要暂停ntp,chrony,openntpd

```shell
sudo service ntp stop
sudo service chrony stop
sudo service openntpd stop
```

开启启动NTPsec服务器

```shell
sudo ./build/main/ntpd/ntpd -n -d -c ntp.conf
```

## 参考链接

-   [https://blog.cloudflare.com/nts-is-now-rfc/](https://blog.cloudflare.com/nts-is-now-rfc/)