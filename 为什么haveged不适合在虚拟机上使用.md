---
title: 为什么haveged不适合在虚拟机上使用？
description: 
published: true
date: 2023-04-30T16:37:16.042Z
tags: haveged
editor: markdown
dateCreated: 2023-04-30T16:37:16.042Z
---



我们先来了解一下haveged是什么？HAVEGE 全称(Hardware Volatile Entropy Gathering and Expansion) ，可以翻译作**硬件挥发性熵收集和扩展**，本质上是一个软件熵源，通过其算法从一些间接的硬件事件生成高质量的随机序列。在计算机系统中，随机数是由熵源产生，而熵源是指具有不确定性的物理或逻辑过程。

**那么为什么虚拟机不适合haveged呢？**由于虚拟机其操作系统无法直接访问宿主机的物理设备，没有足够的随机事件产生高质量的随机序列，当haveged作为熵源时，在被用于生成密钥、签名或者加密数据等任务时可能会造成安全风险。

根据[linux - Is it appropriate to use haveged as a source of entropy on virtual machines? - Information Security Stack Exchange](https://security.stackexchange.com/questions/34523/is-it-appropriate-to-use-haveged-as-a-source-of-entropy-on-virtual-machines)这篇文章中的讨论，虚拟机假如需要获取高质量的随机序列，需要借助`rdtsc`指令发挥作用，而对虚拟机系统来说，使用该指令有三种选择：

- 让cpu指令透传到虚拟机上
- 捕获该指令
- 完全禁用该指令

回到主题，为什么虚拟机不适合haveged？其中争议最大的点，主要围绕安全风险的考量，赞成的人是认为在虚拟机的场景下，没有使用rdtsc指令的情况下的，低质量的随机序列增加的安全风险，而一些使用[haveged - Online Testing (issihosts.com)](http://www.issihosts.com/haveged/ais31.html)一文中的测试数据，认为其安全风险是危言耸听。

我个人觉得，其实Linux内核对随机数生成算法，还是比较重视的，从v5.17可能就开始启用`crng`增强随机数生成器的安全性。