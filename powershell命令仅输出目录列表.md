---
title: powershell命令仅输出目录列表
description: 
published: true
date: 2022-07-09T08:12:51.386Z
tags: powershell
editor: markdown
dateCreated: 2022-06-23T09:25:11.290Z
---

**简介：** powershell命令仅输出目录列表

## powershell命令仅输出目录列表

## 大于powershell 3.0版本可以使用Get-Item、ls、dir、gci

### Get-Item

```
Get-ChildItem -Directory
Get-ChildItem "$path" |  where {$_.Attributes -match'Directory'}
Get-ChildItem "$path" -attributes D -Recurse
```

### ls(alias)

### dir

## 小于powershell 3.0版本

```
Get-ChildItem -Recurse | ?{ $_.PSIsContainer }
```

如果你想要目录的原始字符串名称，你可以这么做

```
Get-ChildItem -Recurse | ?{ $_.PSIsContainer } | Select-Object FullName
```
