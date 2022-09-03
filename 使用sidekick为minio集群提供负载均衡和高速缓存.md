---
title: 使用sidekick为minio集群提供负载均衡和高速缓存
description: 
published: true
date: 2022-09-03T07:27:17.770Z
tags: minio, sidekick, loadbalancing
editor: markdown
dateCreated: 2022-09-03T07:25:24.757Z
---

现在很多云原生应用程序都是用http作为主要的传输机制，但是为web应用程序构建的负载均衡却不能满足一些高性能的场景。如nginx，haproxy虽然能够处理负载的应用场景，但是让它们去支撑一些高性能和一些数据密集型工作，却不能很好的应用。

在minio cluster虽然可以使用nginx作为负载均衡，但是性能在一些高性能场景下很容易达到瓶颈，因此我选择sidekick作为minio cluster作为负载均衡器。sidekick具有下述特性：

- 健康检查，由/v1/health路径提供，能够更好的检测节点的故障
- 能够为S3对象存储提供缓存。
- 简单的层级结构
- 性能有保障

在裸设备配置缓存，先下载sidekick二进制文件

```
wget https://github.com/minio/sidekick/releases/latest/download/sidekick-linux-amd64
```

`````
cd /cache
mv sidekick-linux-amd64 sidekick
chmod +x sidekick
`````

开始配置缓存信息

```
export SIDEKICK_CACHE_ENDPOINT="http://172.168.50.5:9000"
export SIDEKICK_CACHE_ACCESS_KEY="minio"
export SIDEKICK_CACHE_SECRET_KEY="miniodev"
export SIDEKICK_CACHE_BUCKET="cache"
export SIDEKICK_CACHE_MIN_SIZE=64MB
export SIDEKICK_CACHE_HEALTH_DURATION=20
```

我的集群主要由storage0{1...4}组成，因此开启sidekick执行下述命令（注意：是三个点，不然不能被识别）

```
./sidekick --health-path=/minio/health/ready http://storage0{1...4}:9000
```

