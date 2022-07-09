---
title: 安装terraform-ovirt插件为ovirt提供自动化管理
description: 
published: true
date: 2022-07-09T08:13:28.109Z
tags: install, ovirt, terraform
editor: markdown
dateCreated: 2022-06-23T09:27:52.665Z
---

**简介：** 安装terraform-ovirt插件为ovirt提供自动化管理

# ubuntu 18.04|20.04 安装 terrafom ovirt 插件

## 安装golang

先安装go 1.16.15版本，国内下载地址可以通过[Go下载 - Go语言中文网 - Golang中文社区 (studygolang.com)](https://studygolang.com/dl)访问下载

```
apt install -y wget
wget https://studygolang.com/dl/golang/go1.16.15.linux-amd64.tar.gz
```

解压至/usr/local

```
rm -rf /usr/local/go && tar -C /usr/local -xzf go1.16.15.linux-amd64.tar.gz
```

添加环境变量

```
export PATH=$PATH:/usr/local/go/bin
```

查看版本信息

没问题后我们开始安装terraform

## 安装terraform

1.  安装wget curl unzip  
    

```
sudo apt install wget curl unzip
```

2.  查看最新版本并下载  
    

```
TER_VER=`curl -s https://api.github.com/repos/hashicorp/terraform/releases/latest | grep tag_name | cut -d: -f2 | tr -d \"\,\v | awk '{$1=$1};1'`
wget https://releases.hashicorp.com/terraform/${TER_VER}/terraform_${TER_VER}_linux_amd64.zip
```

3.  解压并放到/usr/local/bin  
    

```
$ unzip terraform_${TER_VER}_linux_amd64.zip
Archive:  terraform_xxx_linux_amd64.zip
 inflating: terraform
$ sudo mv terraform /usr/local/bin/
```

4.  查看版本  
    

```
root@terraform:~
Terraform v1.1.7
on linux_amd64
```

## 编译安装terraform-ovirt插件

1.  拉取仓库  
    

```
mkdir -p $HOME/terraform-providers/
cd $HOME/terraform-providers/
git clone https://github.com/oVirt/terraform-provider-ovirt.git
```

2.  配置国内源  
    

```
go env -w GO111MODULE=on
go env -w GOPROXY=https://goproxy.cn,direct
go env | grep GOPROXY
```

3.  开始编译  
    

```
cd terraform-providers/
make build
cp terraform-provider-ovirt \$GOPATH/bin/
```
