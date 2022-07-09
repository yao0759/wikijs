---
title: Linux指定行插入内容的方法总结
description: 
published: true
date: 2022-07-09T08:11:42.605Z
tags: 文本处理, awk, sed
editor: markdown
dateCreated: 2022-06-23T07:02:45.083Z
---

**简介：** Linux指定行插入内容的方法总结

示例文件

```
[root@*** ~] cat FILE 
Line 1
Line 2
Line 3
Line 4
Line 5
Line 6
Line 7
Line 9
```

## 使用sed插入行

```
sed -i '8iLine\ 8' FILE
[root@*** ~] cat FILE 
Line 1
Line 2
Line 3
Line 4
Line 5
Line 6
Line 7
Line 8
Line 9
```

使用上述命令可以在文本中的第8行中插入**Line 8**  

## 使用awk插入行

### 输出到一个新的文件下

```
awk -v n=8 -v s="Line 8" 'NR == n {print s} {print}' FILE > FILE.new
```

### 直接插入

```
awk 'NR==8{print "Line 8"}1' FILE
```

## 使用head tail命令

```
{ head -n 7 FILE; echo "Line 8"; tail -n +8 FILE; } > FILE.new
```

## 使用perl

```
perl -p -e 'print "Line 8\n" if $. == 8' FILE
```
