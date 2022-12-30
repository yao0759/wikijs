---
title: ovirt如何使用命令行管理用户和用户组
description: 
published: true
date: 2022-12-30T15:14:40.152Z
tags: ovirt
editor: markdown
dateCreated: 2022-12-30T15:14:40.152Z
---

**创建用户**

这里我们可以使用ovirt-aaa-jdbc-tool user --help查看配置方法

```
[root@localhost ~]# ovirt-aaa-jdbc-tool user --help
Picked up JAVA_TOOL_OPTIONS: -Dcom.redhat.fips=false
Usage: /usr/bin/ovirt-aaa-jdbc-tool [options] user module ...
Perform user related tasks.
Options:
  --help
    Show help for this module.
Modules:
  add
  edit
  delete
  unlock
  password-reset
  show
  help
See: /usr/bin/ovirt-aaa-jdbc-tool [options] user module --help for help on a specific user module.
```

示例：

先创建用户

```
[root@localhost ~]# ovirt-aaa-jdbc-tool user add test
Picked up JAVA_TOOL_OPTIONS: -Dcom.redhat.fips=false
adding user test...
user added successfully
Note: by default created user cannot log in. see:
/usr/bin/ovirt-aaa-jdbc-tool user password-reset --help.
```

配置密码

```
[root@localhost ~]# ovirt-aaa-jdbc-tool user password-reset test
Picked up JAVA_TOOL_OPTIONS: -Dcom.redhat.fips=false
Password:
```

密码默认需要6个字符以上，默认过期期限是180天，可以通过ovirt-aaa-jdbc-tool settings show查看

```
[root@localhost ~]# ovirt-aaa-jdbc-tool settings show
Picked up JAVA_TOOL_OPTIONS: -Dcom.redhat.fips=false
-- setting --
name: MIN_LENGTH
value: 6
type: class java.lang.Integer
description: passwords are at least X characters long
-- setting --
name: INTERVAL_HOURS
value: 24
type: class java.lang.Integer
...............................................
```

配置完成后，我们可以查看用户信息

```
[root@localhost ~]# ovirt-aaa-jdbc-tool user show test
Picked up JAVA_TOOL_OPTIONS: -Dcom.redhat.fips=false
-- User test(8bc566e9-cd2b-437c-b81a-7379cd426d6c) --
Namespace: *
Name: test
ID: 8bc566e9-cd2b-437c-b81a-7379cd426d6c
Display Name:
Email:
First Name:
Last Name:
Department:
Title:
Description:
Account Disabled: false
Account Locked: false
Account Unlocked At: 1970-01-01 00:00:00Z
Account Valid From: 2022-10-21 22:59:51Z
Account Valid To: 2222-10-21 22:59:51Z
Account Without Password: false
Last successful Login At: 1970-01-01 00:00:00Z
Last unsuccessful Login At: 1970-01-01 00:00:00Z
Password Valid To: 1970-01-01 00:00:00Z
```

我们也可以修改这些设置，如修改所有用户的默认登录会话时长，将其设置为60分钟

```
ovirt-aaa-jdbc-tool settings set --name=MAX_LOGIN_MINUTES --value=60
```

也可以更新多少次登录失败会被锁定

```
ovirt-aaa-jdbc-tool settings set --name=MAX_FAILURES_SINCE_SUCCESS --value=3
```

假如没用了可以选择删除

```
[root@localhost ~]# ovirt-aaa-jdbc-tool user delete test
Picked up JAVA_TOOL_OPTIONS: -Dcom.redhat.fips=false
deleting user test...
user deleted successfully
```

接下来是创建用户组，方法是相似的

```
[root@localhost ~]# ovirt-aaa-jdbc-tool group add group
Picked up JAVA_TOOL_OPTIONS: -Dcom.redhat.fips=false
adding group group...
```

将用户test分配到组group

```
[root@localhost ~]# ovirt-aaa-jdbc-tool group-manage useradd group --user=test
Picked up JAVA_TOOL_OPTIONS: -Dcom.redhat.fips=false
updating user group...
user updated successfully
```

查看用户组信息

```
[root@localhost ~]# ovirt-aaa-jdbc-tool group show group
Picked up JAVA_TOOL_OPTIONS: -Dcom.redhat.fips=false
-- Group group(a0509bb1-d327-4ee4-9c90-7e5cfcd830b1) --
Namespace: *
Name: group
ID: a0509bb1-d327-4ee4-9c90-7e5cfcd830b1
Display Name:
Description:
```

需要的话，甚至可以创建嵌套组，这里演示一下，我新建一个组并将其添加到刚才添加的组内

```
[root@localhost ~]# ovirt-aaa-jdbc-tool group add group-1
Picked up JAVA_TOOL_OPTIONS: -Dcom.redhat.fips=false
adding group group-1...
group added successfully
[root@localhost ~]# ovirt-aaa-jdbc-tool group-manage groupadd group --group=group-1
Picked up JAVA_TOOL_OPTIONS: -Dcom.redhat.fips=false
updating group group...
group updated successfully
```

创建完后，我们可以查看系统到底有哪些用户和用户组

```
[root@localhost ~]# ovirt-aaa-jdbc-tool query --what=group
Picked up JAVA_TOOL_OPTIONS: -Dcom.redhat.fips=false
-- Group group(a0509bb1-d327-4ee4-9c90-7e5cfcd830b1) --
Namespace: *
Name: group
ID: a0509bb1-d327-4ee4-9c90-7e5cfcd830b1
Display Name:
Description:
[root@localhost ~]# ovirt-aaa-jdbc-tool query --what=user
Picked up JAVA_TOOL_OPTIONS: -Dcom.redhat.fips=false
-- User (820c7eed-882a-409a-acdb-b44ff9822113) --
Namespace: *
Name:  
ID: 820c7eed-882a-409a-acdb-b44ff9822113
Display Name:
Email:
First Name:  
Last Name: **
Department:
Title:
Description:
Account Disabled: false
Account Locked: false
Account Unlocked At: 1970-01-01 00:00:00Z
Account Valid From: 2022-09-22 14:12:33Z
Account Valid To: 2222-09-22 14:12:33Z
Account Without Password: false
Last successful Login At: 2022-10-10 14:15:25Z
Last unsuccessful Login At: 2022-10-10 14:14:26Z
Password Valid To: 2035-08-01 12:00:00Z
```