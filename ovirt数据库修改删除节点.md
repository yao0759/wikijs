---
title: ovirt数据库修改删除节点
description: 
published: true
date: 2022-07-09T08:12:43.390Z
tags: ovirt, postgresql
editor: markdown
dateCreated: 2022-06-23T09:57:52.987Z
---

**简介：** ovirt数据库修改删除节点

ovirt遇到一个问题，因为没有在ovirt engine中删除就重装了其中一个节点，导致重新添加该节点时出现了一个报错“**Host with the same UUID already exists.**”

因此需要进入数据库中删除节点信息

登录ovirt engine输入

```
su postgre
psql -s engine
```

然后执行

```
select vds_id from vds_static where host_name = 'HOST_NAME';
delete from vds_statistics where vds_id = 'id';
delete from vds_dynamic where vds_id = 'id';
delete from vds_static where vds_id = 'id';
```
