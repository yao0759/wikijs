---
title: openzfs draid是什么
description: 
published: true
date: 2022-07-09T08:12:35.843Z
tags: openzfs, draid
editor: markdown
dateCreated: 2022-06-23T12:37:01.897Z
---

=

**简介：** openzfs draid是什么

openzfs在2.1版本中引入了一个新功能叫draid，它跟raidz有什么区别吗？来讨论一下这个功能特性的意义。

一般来说，raid磁盘阵列的磁盘空间越大，重建所需要的时间也越长，尤其是使用了raid6的情况下。但是在重建过程中还是存在数据丢失的风险，因此各家存储供应商都拥有自己的raid重建机制，其目的是为了更快的重建磁盘，加快这一速度。

因此早在2010年时，就有人提出将奇偶校验分散可以加快磁盘重建速度，让所有的磁盘都参与到重建的过程中，而不仅仅是受影响的磁盘。

具体的差异可以参考openzfs给出的参考图

![image.png](https://ucc.alicdn.com/pic/developer-ecology/b0abfec11a9f4446a0b0b46c79544505.png "image.png")

同时draid跟原先的raidz不同之处还在于它使用固定的条带宽度，无论存储对象需要多少个blocks，都是一次性分配的。但这样也意味着它会在一定程度上影响可用容量和IOPS。

下面是draid和raidz重建速度的对比

![image.png](https://ucc.alicdn.com/pic/developer-ecology/3d6f19933c6646bab7123031ced0d26c.png "image.png")
