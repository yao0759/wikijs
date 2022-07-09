---
title: gitea配置全局代理用于镜像github源
description: 
published: true
date: 2022-07-09T08:12:09.407Z
tags: git, gitea
editor: markdown
dateCreated: 2022-06-23T12:35:01.619Z
---

**简介：** gitea配置全局代理用于镜像github源

gitea打算加入代理功能方便mirror一些仓库，因为国内访问速度不佳

只需要添加下述配置到config/app.ini

```
......
PROXY_ENABLED = true
PROXY_URL = socks://127.0.0.1:1080
PROXY_HOSTS = *.github.com
......
```
