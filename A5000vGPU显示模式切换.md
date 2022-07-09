---
title: A5000 vGPU显示模式切换
description: 
published: true
date: 2022-07-09T08:11:18.981Z
tags: vgpu, 显卡, 硬件
editor: markdown
dateCreated: 2022-06-23T06:58:53.413Z
---

# A5000 vGPU显示模式切换
## 原因
最近虚拟化服务器要新增两块A5000，用于分配vGPU，插入后用**lspci -vvv | grep NVI**查看发现输出信息跟之前的不一样，带有音频接口，而且无法通过**/usr/lib/nvidia/sriov**启用VF。后来想起来，A5000要作为vGPU分配要切换显卡模式。

lspci输出信息如下图所示

![image.png](https://ucc.alicdn.com/pic/developer-ecology/c4fd31efa5444b47b5b544310dada0a4.png "image.png")
## 解决办法

下载工具[**nvidia display mode selector tool**](https://developer.nvidia.com/displaymodeselector)

然后在server端解压执行**./displaymodeselector --gpumode compute --auto，**这样就能把A5000显卡从graphics切换到computer**（该方法同样适用于A6000和A40）**。

切换后重启，再次使用**lspci -vvv | grep NVI查看**

![image.png](https://ucc.alicdn.com/pic/developer-ecology/70e108b7b2b34fd296bd1a9752a05740.png "image.png")