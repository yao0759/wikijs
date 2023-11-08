---
title: 开源项目分享--sshx
description: 
published: true
date: 2023-11-08T15:00:25.985Z
tags: sshx
editor: markdown
dateCreated: 2023-11-08T15:00:25.984Z
---



sshx 可让你在一个多人的无限画布上，通过链接与任何人共享终端。

它具有实时协作、远程光标和聊天功能。它还采用 Rust 编写的轻量级服务器，速度快、端到端加密。

我们可以通过官方提供的连接给mac或者Linux下载安装客户端

```
curl -sSf https://sshx.io/get | sh
```

安装完成执行，只需要执行`sshx`就会弹出下述信息

```
  sshx v0.2.0

  ➜  Link:  https://sshx.io/s/mBJTMFi9kM#T3doWegeoGsnyw
  ➜  Shell: /usr/bin/zsh
```

然后我们可以通过提供的链接访问ssh了，进入后让你输入一个别名，这个用户名用于多人协作的同步

![image-20231108224555989](https://yhblog-1254039996.cos.ap-guangzhou.myqcloud.com/img-blog/image-20231108224555989.png)

输入名称后，就可以看到，看到这样一个界面，默认是一个黑色背景网格，上面还有些工具栏，作用分别是创建terminal，聊天窗口，设置和网络延迟状况，这里我们基本只需要用到创建terminal

![image-20231108224700843](https://yhblog-1254039996.cos.ap-guangzhou.myqcloud.com/img-blog/image-20231108224700843.png)

点击创建terminal后，会如图所示，这里我多创建几个窗口，可以堆在一起了，这个时候画布的好处就出现了，我们可以无限延展，同时创建的terminal会保留默认的shell环境，如这里我默认的shell是zsh，可以看到还是一样有保留的。

![image-20231108225039229](https://yhblog-1254039996.cos.ap-guangzhou.myqcloud.com/img-blog/image-20231108225039229.png)

这里我还测试了X11的功能，我执行`xclock`发现一个有意思的现象，虽然它无法弹出对应的窗口，但是我的屏幕其实弹出的了，可以看下述图示

![image-20231108225409023](https://yhblog-1254039996.cos.ap-guangzhou.myqcloud.com/img-blog/image-20231108225409023.png)

![image-20231108225422385](https://yhblog-1254039996.cos.ap-guangzhou.myqcloud.com/img-blog/image-20231108225422385.png)

我们再看一下多人协助的部分，我新建了一个无痕窗口访问，可以看到当有人操作时，鼠标的指针其实也会同步，显示有谁在操作，在输入信息时，也会显示该窗口是谁在输入，很有趣

![image-20231108225607645](https://yhblog-1254039996.cos.ap-guangzhou.myqcloud.com/img-blog/image-20231108225607645.png)

![image-20231108225709491](https://yhblog-1254039996.cos.ap-guangzhou.myqcloud.com/img-blog/image-20231108225709491.png)