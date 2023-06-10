---
title: discourse安装plugin方法
description: 
published: true
date: 2023-06-10T04:01:14.988Z
tags: discourse
editor: markdown
dateCreated: 2022-12-30T15:13:14.281Z
---

最近需要安装discourse-assign和tickets-plugin两个插件到discourse上。记录一下安装步骤

移动到discourse目录下

```
cd /var/discourse
```

修改app.xml，移动到hooks下

```bash
vim containers/app.yml
......
hooks:
  after_code:
    - exec:
        cd: $home/plugins
        cmd:
          - git clone https://github.com/discourse/docker_manager.git
          - git clone https://github.com/paviliondev/discourse-tickets.git
          - git clone https://github.com/discourse/discourse-assign.git
......
```

重建应用程序

```
./launcher rebuild app
```

然后等待重建完成后，进入设置界面开启这两个功能即可