---
title: guacamole修改url路径及登录页面上的logo和title name
description: 
published: true
date: 2022-07-09T08:12:24.854Z
tags: guacamole
editor: markdown
dateCreated: 2022-06-23T10:00:14.147Z
---

**简介：** guacamole修改url路径及登录页面上的logo和title name

## 修改url路径

备份目录及文件

```
cd /var/lib/tomcat9/webapps/
cp -r guacamole /root/guacamole_backup/
cp guacamole.war /root/guacamole_backup/
```

暂停服务

```
systemctl stop tomcat9
systemctl stop guacd
```

修改目录及文件名为你需要的url路径名如test

```
cd /var/lib/tomcat9/webapps/
mv guacamole test
mv guacamole.war test.war
```

## 修改logo

将希望替换的图片文件名修改成guac-tricolor.png，然后替换/var/lib/tomcat9/webapps/guacamole/images/guac-tricolor.png

```
cp /home/ubuntu/guac-tricolor.png /var/lib/tomcat9/webapps/guacamole/images/
```

## 修改title name

```
vim /var/lib/tomcat9/webapps/guacamole/translations/en.json
......
"NAME"    : "test",
......
```

