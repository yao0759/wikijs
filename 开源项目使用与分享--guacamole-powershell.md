---
title: 开源项目使用与分享--guacamole-powershell
description: 
published: true
date: 2022-12-30T15:09:50.841Z
tags: powershell, guacamole
editor: markdown
dateCreated: 2022-12-30T15:09:50.841Z
---

最近在环境中用了guacamole去访问一些vnc桌面或者rdp桌面，且连接的桌面和用户数不少，数量一多，问题就出现了。guacamole本身的api对于我来说，使用上有瓶颈，有没有一个易用且能够解决我在工作中的批量创建和管理的问题呢？我恰好在github上找到这个项目[UpperM/guacamole-powershell: PowerShell functions useful to manage Apache Guacamole](https://github.com/UpperM/guacamole-powershell)；一个powershell项目，但是出奇的易用。

## 安装

下载包，并解压，建议文件放在**C:\Windows\system32\WindowsPowerShell\v1.0\Modules\**

修改策略

```
Set-ExecutionPolicy RemoteSigned
```

导入module

```
Import-Module -Name C:\Windows\system32\WindowsPowerShell\v1.0\Modules\guacamole-powershell-master\PSGuacamole
```

导入成功后，我们就能开始配置

## 配置

获取令牌（guacamole后面不用反斜线，不然会出问题）

```
New-GuacToken -Username "admin" -Password "MyPassword" -Server "http://srv-guacamole:8080/guacamole"
```

认证完成后，我们就能开始做一些常用的操作了，如创建用户，创建连接，检查连接，这部分可以直接查看readme[[UpperM/guacamole-powershell: PowerShell functions useful to manage Apache Guacamole (github.com)\]](https://github.com/UpperM/guacamole-powershell)

具体功能包括了：

- Get-GuacConnectionsGroup
- Get-GuacConnectionsGroupDetails
- Get-GuacConnectionsGroups
- Get-GuacConnectionsGroupsConnections
- New-GuacConnectionGroup
- Remove-GuacConnectionGroup
- Update-GuacConnectionGroup
- Get-GuacActiveConnections
- Get-GuacConnection
- Get-GuacConnections
- New-GuacConnection
- Remove-GuacConnection
- Stop-GuacConnection
- Update-GuacConnection
- Add-GuacUserGroupConnection
- Add-GuacUserGroupMember
- Add-GuacUserGroupPermission
- Get-GuacUserGroup
- Get-GuacUserGroups
- New-GuacUserGroup
- Remove-GuacUserGroup
- Remove-GuacUserGroupConnection
- Remove-GuacUserGroupPermission
- Update-GuacUserGroup
- Update-GuacUserGroupParent
- ...........