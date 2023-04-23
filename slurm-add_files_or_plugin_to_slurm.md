---
title: slurm--为Slurm添加文件或插件
description: 
published: true
date: 2023-04-23T15:21:15.386Z
tags: slurm
editor: markdown
dateCreated: 2023-04-23T15:21:15.386Z
---

## 添加文件到slurm
这是为了在Slurm代码库中添加新的C文件而需要遵循的程序。我们建议为此目的使用一个git分支。

1. 将你的新文件添加到git仓库中。
2. 修改该文件父目录下的 "Makefile.am "文件。
3. 在Slurm的顶层目录下执行 "autoreconf"。如果你有老版本的autoconf、automake、libtool或aclocal，那么你可能需要手动修改文件父目录下的Makefile.in文件。如果你有与Slurm团队最初使用的不同版本的文件，这可能会重建Slurm中所有的Makefile.in文件

## 添加plugin到slurm

这是在Slurm代码库中添加新插件时需要遵循的程序。我们推荐使用git分支来实现这一目的。在这个例子中，我们展示了为了添加一个名为 "topology/4d_torus "的插件，需要修改哪些文件。

1. 为这个插件创建一个新目录（例如 "src/plugins/topology/4d_torus"）。
2. 将这个新目录添加到其父目录的 "Makefile.am "文件中（例如，"src/plugins/topology/Makefile.am"）。
3. 将你的新文件放到适当的目录中（例如，"src/plugins/topology/4d_torus/topology_4d_torus.c"）
4. 在新目录下创建一个 "Makefile.am "文件，标识新文件（例如 "src/plugins/topology/4d_torus/Makefile.am"）。使用一个现有的 "Makefile.am "文件作为模型。
5. 在 "configure.ac "文件中确定要在Slurm配置时构建的新Makefile。请保持条目的英文字母顺序。
6. 在Slurm的顶层目录中执行 "autoreconf"。如果你有旧版本的autoconf、automake、libtool或aclocal，那么你可能需要手动创建或修改Makefile.in文件。如果你有与Slurm团队最初使用的不同版本的文件，这可能会重建Slurm中所有的Makefile.in文件。
7. 修改 "slurm.spec "文件，在适当的RPM中包括新的插件文件。
8. 将新的文件，包括 "Makefile.am "和 "Makefile.in"，添加到git仓库。