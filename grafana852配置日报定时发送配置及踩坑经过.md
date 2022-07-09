---
title: grafana 8.x配置日报定时发送配置及踩坑经过
description: 
published: true
date: 2022-07-09T08:12:17.314Z
tags: grafana
editor: markdown
dateCreated: 2022-06-23T12:34:20.627Z
---

2022-05-14 125

**简介：** grafana 8.x配置日报定时发送配置及踩坑经过

## grafana 8.x配置日报定时发送配置及踩坑经过

## 环境说明

系统版本：CentOS 7.9

Grafana版本：8.5.2

## 配置说明

想要每日定时发送系统运行状态给leader和运维管理人员查看，因为开源版本并不具备enterprise那样拥有reporting功能，假如我们需要实现类似的功能，需要依靠[IzakMarais/reporter: Service that generates a PDF report from a Grafana dashboard (github.com)](https://github.com/IzakMarais/reporter)这个项目去实现。

## 安装

### 安装grafana-reporter

安装texlive包和go

```
yum install go git
yum install texlive-pdftex texlive-latex-bin texlive-texconfig* texlive-latex* texlive-metafont* texlive-cmap* texlive-ec texlive-fncychap* texlive-pdftex-def texlive-fancyhdr* texlive-titlesec* texlive-multirow texlive-framed* texlive-wrapfig* texlive-parskip* texlive-caption texlive-ifluatex* texlive-collection-fontsrecommended texlive-collection-latexrecommended texinfo-tex
```

获取grafana-reporter的源码和依赖包

```
go get github.com/IzakMarais/reporter/…
```

编译安装二进制文件

```
go install github.com/IzakMarais/reporter/cmd/grafana-reporter@latest
```

编译完成后，会在go/bin/下生成grafana-reporter二进制文件

添加服务

```
cat > /etc/systemd/system/grafana-reporter.service << EOF
[Unit]
Description=Grafana Reporter
After=grafana-reporter.service
[Service]
Type=sample
ExecStart=/root/go/bin/grafana-reporter -ip localhost:3000
ExecStop=pkill -9 grafana-report
StandardOutput=syslog
StandardError=syslog
[Install]
WantedBy=multi-user.target
EOF
```

配置开机自启动

```
systemctl daemon-reload
systemctl enable --now grafana-reporter.service
```

防火墙开放8686端口

```
firewall-cmd --zone=public --permanent --add-port=8686/tcp
firewall-cmd --reload
```

### 开始配置grafana links

需要安装Grafana Image Renderer插件，不然可能出现下图所示的错误

```
grafana-cli plugins install grafana-image-renderer
```

![1.png](https://ucc.alicdn.com/pic/developer-ecology/2eb41b6582db4b02b368d89380225f18.png "1.png")

重启服务

```
systemctl restart grafana-server
```

添加API Keys,roles选择Viewer即可。

![2.png](https://ucc.alicdn.com/pic/developer-ecology/f952ee61040641969f0a2d95e54e2156.png "2.png")

生成后，会显示Key，点击Copy，复制下来Key，准备添加到links配置中

选中需要daily report的dashboard，点击设备按钮开始配置

![3.png](https://ucc.alicdn.com/pic/developer-ecology/d0003083349c4c3db690b96ff15710f5.png "3.png")

![4.png](https://ucc.alicdn.com/pic/developer-ecology/b14d0a960d2d464da15538aea21bbe49.png "4.png")

![5.png](https://ucc.alicdn.com/pic/developer-ecology/ffb137b8f9ef4c73822314d397fc93b9.png "5.png")

URL这里填写http://<grafana-access-ip>:3000/api/v5/report/<dashboard-path>?apitoken=<key-create>

![6.png](https://ucc.alicdn.com/pic/developer-ecology/8abb13eac3c24ba886ca2b019cadfce2.png "6.png")

dashboard-path路径可以从这里查看

 ![7.png](https://ucc.alicdn.com/pic/developer-ecology/a21e08c7ac5040dea3019b538574fbeb.png "7.png")完成后，回到dashboard，点解report查看是否生效

![8.png](https://ucc.alicdn.com/pic/developer-ecology/6e9bbdbbf6cb4475b956a2b68b8bb274.png "8.png")

假如遇到了下述的问题，那可能是缺少一些依赖包

![10.png](https://ucc.alicdn.com/pic/developer-ecology/352de3e88a5f45d98d6d6d955f6e7ea0.png "10.png")

我们可以从/var/log/grafana/grafana.log查看到底缺少哪些包，如下图所示

![9.png](https://ucc.alicdn.com/pic/developer-ecology/c27c967064d74c1fb450420e6f4cfbc1.png "9.png")

解决依赖问题

```
yum install atk -y
yum install at-spi2-atk -y
yum -y install cups-libs
yum install libXss* -y
yum install libX11
yum install -y libXcomposite libXcomposite-devel
```

假如日志没有其他报错后，应该是可以生成pdf文件了

### 开始配置定时发送邮件

安装mail

修改/etc/mail.rc，对该文件追加下述内容，因为我用的是qq企业邮箱

```
cat >> /etc/mail.rc << EOF
set from=<mail>
set smtp=smtps://smtp.exmail.qq.com:465
set smtp-auth-user=<mail>
set smtp-auth-password=wbxMo9q9gaUWej5Z
set smtp-auth=login
set ssl-verify=ignore
set nss-config-dir=/etc/pki/nssdb/
EOF
```

接下来还需要写个脚本自动发送。

