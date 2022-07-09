---
title: grafana怎么读取ganglia的rrd展示到dashboard中
description: 
published: true
date: 2022-07-09T08:12:21.188Z
tags: ganglia, grafana
editor: markdown
dateCreated: 2022-06-23T12:30:39.350Z
---

**简介：** grafana怎么读取ganglia的rrd展示到dashboard中

## 环境

-   ganglia服务器 + grafana 服务器在同一台机器上 ，系统版本为centos7

## 原因

想要将ganglia中数据放到grafana中展示，但是没有找到什么好的方法。但有人提到可以使用这个项目实现[https://github.com/doublemarket/grafana-rrd-server](https://github.com/doublemarket/grafana-rrd-server)，一个简单的HTTP服务器，可以读取RRD文件并响应来自Grafana的请求与Grafana简单JSON数据源插件。有类似需求可能不少，但是相应的方法记录比较少，因此觉得分享一下我的方法。

## 安装

### 安装依赖

安装golang

安装librrd

```
yum install rrdtool-devel
```

### 开始安装

```
go get github.com/doublemarket/grafana-rrd-server
```

### 添加服务

```
useradd grafanarrd
cat > /etc/systemd/system/grafana-rrd-server.service <<EOF
[Unit]
Description=Grafana RRD Server
After=network.service

[Service]
User=grafanarrd
Group=grafanarrd
Restart=on-failure
Environment="LD_LIBRARY_PATH=/opt/rrdtool-1.6/lib"
ExecStart=/opt/grafana-rrd-server/grafana-rrd-server -p 9000 -r /path/to/rrds -s 300
RestartSec=10s

[Install]
WantedBy=default.target
EOF
```

```
systemctl daemon-reload
systemctl enable grafana-rrd-server
systemctl start grafana-rrd-server
```

### grafana安装grafana-simple-json-datasource插件

```
grafana-cli plugins install grafana-simple-json-datasource
```

### 添加数据源

如图所示

![image-20220511220152818.png](https://ucc.alicdn.com/pic/developer-ecology/29e206e5606b470bad0fa95510fef921.png "image-20220511220152818.png")

然后就可以添加dashboard了，实现的效果大概是这样

![image-20220511220107298.png](https://ucc.alicdn.com/pic/developer-ecology/3774bf916342454aa0de10207803a609.png "image-20220511220107298.png")

其实这样实现其实还是有不少问题的，如数据值不是很好阅读，还有一些数据无法争取读取，问题不少，仅作为一个参考探讨一下而已。

