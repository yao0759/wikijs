---
title: 如何通过/proc/net/fib_trie获取ip地址
description: 
published: true
date: 2023-06-05T13:22:21.600Z
tags: proc
editor: markdown
dateCreated: 2023-06-05T13:22:21.600Z
---


## 参考链接

- [The IPv4 Routing Subsystem - Linux Kernel Networking: Implementation and Theory (2014) (apprize.best)](https://apprize.best/linux/kernel/6.html)

## 正文

当我们使用`cat /proc/net/fib_trie`，会得到下述信息

```bash
# cat /proc/net/fib_trie
Main:
  +-- 0.0.0.0/0 3 0 4
     +-- 0.0.0.0/4 2 0 2
        |-- 0.0.0.0
           /0 universe UNICAST
        +-- 10.8.0.0/13 2 0 2
           +-- 10.8.0.0/24 2 0 2
              +-- 10.8.0.0/31 1 0 0
                 |-- 10.8.0.0
                    /32 link BROADCAST
                    /24 link UNICAST
                 |-- 10.8.0.1
                    /32 host LOCAL
              |-- 10.8.0.255
                 /32 link BROADCAST
           +-- 10.13.0.0/16 2 0 1
              |-- 10.13.0.0
                 /32 link BROADCAST
                 /16 link UNICAST
              |-- 10.13.132.171
                 /32 host LOCAL
              |-- 10.13.255.255
                 /32 link BROADCAST
     +-- 127.0.0.0/8 2 0 2
        +-- 127.0.0.0/31 1 0 0
           |-- 127.0.0.0
              /32 link BROADCAST
              /8 host LOCAL
           |-- 127.0.0.1
              /32 host LOCAL
        |-- 127.255.255.255
           /32 link BROADCAST
     |-- 169.254.0.0
        /16 link UNICAST
     +-- 192.168.191.0/24 2 0 2
        |-- 192.168.191.0
           /32 link BROADCAST
           /24 link UNICAST
        +-- 192.168.191.224/27 2 0 2
           |-- 192.168.191.238
              /32 host LOCAL
           |-- 192.168.191.255
              /32 link BROADCAST

```

`/proc/net/fib_trie`文件提供了关于FIB（Forwarding Information Base，转发信息库）Trie（前缀树）的信息。其作用是高效地存储和查找路由表项。它以一种前缀树的形式组织了路由表项，其中每个节点表示一个路由前缀。通过在树中进行前缀匹配，内核可以快速找到与目标IP地址最匹配的路由表项。

![IPv4 route lookup on Linux](https://d2pzklc15kok91.cloudfront.net/images/linux/lpc-trie-patricia-trie-lookup-v3.9e110e0b8371bb.svg)

因此我们可以用下述命令查看ip信息

```bash
awk '/32 host/ { print i } {i=$2}' /proc/net/fib_trie
```

