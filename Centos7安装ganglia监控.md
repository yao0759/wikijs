---
title: Centos7安装ganglia监控
description: 
published: true
date: 2022-07-09T08:11:22.825Z
tags: centos7, install, ganglia
editor: markdown
dateCreated: 2022-06-23T12:29:43.695Z
---

**简介：** Centos7安装ganglia监控

## 参考链接

-   [Ganglia上的gpu监测配置 (Ubuntu) - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/129043146)
-   [How to Install Ganglia on CentOS 7 - slothparadise](https://www.slothparadise.com/how-to-install-ganglia-on-centos-7/)
-   [Setup Real-Time Monitoring using Ganglia on Centos 7 | by Mohammad Hanif | Medium](https://medium.com/@cakhanif/setup-real-time-monitoring-using-ganglia-on-centos-7-45706a49ea89)

## 环境说明

```
ganglia服务器 -- centos7
ganglia客户端 -- ubuntu1804
```

## 安装步骤

### 服务端安装步骤

先安装epel仓库

```
yum install epel-release -y
yum update
```

先关闭selinux

```
sed -i s/^SELINUX=.*$/SELINUX=disabled/ /etc/selinux/config
```

重启，然后开始安装ganglia

```
yum install -y rrdtool rrdtool-devel ganglia-web ganglia-gmetad  ganglia-gmond ganglia-gmond-python httpd httpd-tools apr-devel zlib-devel  libconfuse-devel expat-devel pcre-devel git
```

修改**/etc/ganglia/gmetad.conf**，将"Example Cluster"修改成你需要的集群名字

```
vim /etc/ganglia/gmetad.conf
...
data_source "Example Cluster" master
...
```

紧接着修改**/etc/ganglia/gmond.conf**

```
vim /etc/ganglia/gmond.conf
...
cluster {
name = “Example Cluster”
owner = “my server”
latlong = “unspecified”
url = “unspecified”
}
host {
location = “unspecified”
}
udp_send_channel {
host = master
port = 8649
ttl = 1
}
udp_recv_channel {
port = 8649
}
tcp_accept_channel {
port = 8649
}
...
```

修改**/etc/httpd/conf.d/ganglia.conf**，开放访问

```
vim /etc/httpd/conf.d/ganglia.conf
...
Alias /ganglia /usr/share/ganglia
<Location /ganglia>
  Order deny,allow
  Allow from all
  Require all granted
  # Allow from .example.com
</Location>
...
```

然后开放防火墙

```
firewall-cmd --permanent --zone=public --add-port=8649/udp
firewall-cmd --permanent --zone=public --add-port=8649/tcp
firewall-cmd --permanent --zone=public --add-port=8651/tcp
firewall-cmd --permanent --zone=public --add-port=8652/tcp
firewall-cmd --reload
```

开启服务

```
chkconfig httpd
chkconfig gmetad on
chkconfig gmond on
systemctl start httpd
systemctl start gmetad
systemctl start gmond
```

### 服务端添加nvidia gpu图表

```
git clone https://github.com/ganglia/gmond_python_modules.git
cd gmond_python_modules/gpu/nvidia
cp graph.d/* /usr/share/ganglia/graph.d/
systemctl restart gmetad
```

![image-20211214191524679.png](https://ucc.alicdn.com/pic/developer-ecology/2c63a0c038fb4102854fdfaf494f1ddf.png "image-20211214191524679.png")

### 客户端(ubuntu)装ganglia-client

安装客户端

```
apt install -y ganglia-monitor
```

修改客户端配置文件

```
vim /etc/ganglia/gmond.conf
···
host {
  location = "unspecified"
}
/* Feel free to specify as many udp_send_channels as you like.  Gmond
   used to only support having a single channel */
udp_send_channel {
  host = "IP Address"
  port = 8649
  ttl = 1
}
udp_recv_channel {
  port = 8649
  retry_bind = true
}
tcp_accept_channel {
}
···
```

启动服务

```
/etc/init.d/ganglia-monitor start
```

### 客户端添加nvidia gpu插件

安装nvidia gpu监控插件，由于ubuntu18.04默认使用python3，而ganglia的nvidia gpu插件需要使用python2版本，因此在安装前安装python2 pip2

```
apt install python2-pip python2
```

安装nvidia-ml-py-3.295.00

```
pip install nvidia-ml-py-3.295.00
```

git拉取gmond\_python\_modules并配置

```
git clone https://github.com/ganglia/gmond_python_modules.git
```

创建文件夹及拷贝配置文件

```
mkdir /etc/ganglia/conf.d/
mkdir /usr/lib/ganglia/python_modules/
cd gmond_python_modules/gpu/nvidia
cp conf.d/nvidia.conf /etc/ganglia/conf.d/
cat <<EOF | sudo tee /etc/ganglia/conf.d/modpython.conf
modules {
  module {
    name = "python_module"
    path = "/usr/lib/ganglia/modpython.so"
    params = "/usr/lib/ganglia/python_modules/"
  }
}
include ('/etc/ganglia/conf.d/*.pyconf')
EOF
cp python_modules/nvidia.py /usr/lib/ganglia/python_modules/
```

检验是否pynvml安装成功

```
python /usr/lib/ganglia/python_modules/nvidia.py
```

![image-20220511212743473.png](https://ucc.alicdn.com/pic/developer-ecology/8c266066ae254527badb76b224b607de.png "image-20220511212743473.png")

