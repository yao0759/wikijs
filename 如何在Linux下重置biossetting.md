---
title: 如何在Linux下重置bios setting
description: 
published: true
date: 2022-07-09T08:13:24.363Z
tags: 硬件, bios
editor: markdown
dateCreated: 2022-06-23T12:26:03.814Z
---

**简介：** 如何在Linux下重置bios setting

下述的方法只是探讨为主，不建议在生产环境下使用

今天被问到一个问题，要用命令重置恢复bios默认设置。因为bios是被写在ROM上，无法直接设置，但是大多数服务器主板自带bcm，通过ipmitool说不定就可以了，因此去搜索一些相关文档说不定可以设置，发现了下列的一些信息，粘贴出来参考一下。

[https://docs.oracle.com/cd/E19962-01/html/821-1081/gkezr.html](https://docs.oracle.com/cd/E19962-01/html/821-1081/gkezr.html) Clearing CMOS Settings After an Update (Optional) If you cannot get output to your serial console after the firmware update, you might have to clear CMOS settings. This is because your default CMOS settings might have been changed by the update of the BIOS.

To clear CMOS settings, use the following IPMItool commands (in this example, the default username, root, and the default password, changeme, are used):

```
ipmitool -U root -P changeme -H SP-IP chassis power off
ipmitool -U root -P changeme -H SP-IP chassis bootdev disk clear-cmos=yes
```

后续，或者可以尝试如下方法（具有一定风险性）

```
modprobe nvram
dd if=/dev/zero of=/dev/nvram
```