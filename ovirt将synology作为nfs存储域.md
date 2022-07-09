---
title: ovirt将synology作为nfs存储域
description: 
published: true
date: 2022-07-09T08:12:39.796Z
tags: kvm, ovirt, nfs
editor: markdown
dateCreated: 2022-07-09T07:39:45.895Z
---

# ovirt将synology作为nfs存储域

最近有个东西比较折腾，需要将synology的nfs共享存储添加为存储域，原本以为只是很简单的问题，但是发现添加后总是提示报错。查看synology下的日志提示下述报错( kernel: NFSD: client X.X.X.X testing state ID with incorrect client ID)

![image-20220709151355093](https://yhblog-1254039996.cos.ap-guangzhou.myqcloud.com/img-blog/image-20220709151355093.png)

查询该问题指出该问题有可能是安全软件或者是缺少对应的用户和用户组，恰好ovirt似乎有对应的[解决方法](https://www.ovirt.org/develop/troubleshooting-nfs-storage-issues.html)。

参考上述链接，我们使用ssh登录到synology终端下。

1. 创建kvm group(id 36)

   ```
   echo "kvm:x:36" >> /etc/group
   ```

2. 创建vdsm用户(uid 36)

   ```
   echo "vdsm:x:36:36::/dev/null:/bin/false" >> /etc/passwd
   ```

3. 更新synology ACLs以允许kvm:vsdm访问

   ```
       EXPORT_DIR=/volume1/...
       ALLOW_USER='user:vdsm:allow:rwxpdDaARWcCo:fd--'
       ALLOW_GROUP='group:kvm:allow:rwxpdDaARWcCo:fd--'
   
       synoacltool -get "$EXPORT_DIR" | grep -q "$ALLOW_USER" || synoacltool -add "$EXPORT_DIR" $ALLOW_USER > /dev/null
       synoacltool -get "$EXPORT_DIR" | grep -q "$ALLOW_GROUP" || synoacltool -add "$EXPORT_DIR" $ALLOW_GROUP > /dev/null
       synoacltool -get "$EXPORT_DIR"
   ```

4. 编辑/etc/exports，将需要共享的目录的anonuid和anongid替换为36

   ```
   /volume1/ovirt 1.1.1.1(<Synology defined options>,anonuid=1025,anongid=100) 1.1.1.2(<Synology defined options>anonuid=1025,anongid=100)
   ```

结束这一个步骤后，重新nfs-server服务，然后重新添加还是有问题。

最后发现，在添加域时，需要将nfs的**"自动协商"**修改成**"nfsv3"**

![image-20220709153632509](https://yhblog-1254039996.cos.ap-guangzhou.myqcloud.com/img-blog/image-20220709153632509.png)

