---
title: shell脚本将文本每行内容作为数组
description: 
published: true
date: 2022-12-30T15:11:33.125Z
tags: 文本处理, shell
editor: markdown
dateCreated: 2022-12-30T15:11:33.125Z
---



使用mapfile，也可以使用readarray，两个其实是同一个命令

```
mapfile -t arr < output
```

使用for循环

```
a=0
for line in `cat file.txt`
do
    b=$line
  arr[$a]=$b
  ((a++))
done
```

while循环

```
arr=()

while IFS= read -r line; do
    time+=("$line")
done < output
```