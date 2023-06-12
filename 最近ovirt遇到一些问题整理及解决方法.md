---
title: 最近ovirt遇到一些问题整理及解决方法
description: 
published: true
date: 2023-06-12T05:24:29.300Z
tags: ovirt
editor: markdown
dateCreated: 2022-09-03T07:30:42.213Z
---

## 添加节点显示installfailed

先说添加ovirt节点最近遇到了添加失败，ovirt engine上会显示installfailed，查看日志后发现，有下述信息

```
Error: Failed to download metadata for repo 'ovirt-4.4-centos-gluster8': Cannot prepare internal mirrorlist: No URLs in mirrorlist
```

在失败节点上执行

```
dnf makecache
```

也有类似的报错

```
[root@localhost ~]# dnf makecache
Ceph packages for x86_64                                                                         3.9 kB/s | 3.0 kB     00:00
oVirt Node Optional packages from CentOS Stream 8 - BaseOS                                       6.5 kB/s | 3.9 kB     00:00
oVirt Node Optional packages from CentOS Stream 8 - AppStream                                    5.9 kB/s | 4.4 kB     00:00
Latest oVirt 4.4 Release                                                                         3.9 kB/s | 3.0 kB     00:00
Extra Packages for Enterprise Linux 8 - x86_64                                                    38 kB/s | 8.6 kB     00:00
CentOS-8 - Gluster 8                                                                              69  B/s |  38  B     00:00
Error: Failed to download metadata for repo 'ovirt-4.4-centos-gluster8': Cannot prepare internal mirrorlist: No URLs in mirrorlist
```

由此可知是源的问题，这是因为centos8在2022年后已经停止了，因此需要安装第三方源的ovirt-release，可以使用下述命令进行替换。

```
dnf --disablerepo=ovirt-4.4-centos-gluster8,ovirt-4.4-centos-opstools,ovirt-4.4-openstack-victoria install -y http://mirror.massclouds.com/ovirt/yum-repo/ovirt-release44.rpm
```

## 节点证书创建失败

我在添加节点后还是失败，发现相关证书失败，查看日志显示下述信息

```
...
        "results" : [ {
          "msg" : "non-zero return code",
          "cmd" : [ "/usr/share/ovirt-engine/bin/pki-enroll-request.sh", "--name=10.10.50.138", "--subject=/O=********/CN=10.10.50.138", "--san=IP:10.10.50.138", "--days=398", "--timeout=30", "--ca-file=ca", "--cert-dir=certs", "--req-dir=requests" ],
          "stdout" : "",
          "stderr" : "Using configuration from openssl.conf\nunable to load number from serial.txt\nerror while loading serial number\n140631516194624:error:0D066096:asn1 encoding routines:a2i_ASN1_INTEGER:short line:crypto/asn1/f_int.c:140:\nCannot sign certificate",
          "rc" : 1,
          "start" : "2022-09-03 09:00:49.661484",
          "end" : "2022-09-03 09:00:49.682199",
          "delta" : "0:00:00.020715",
          "changed" : true,
          "failed" : true,
          "invocation" : {
            "module_args" : {
              "_raw_params" : "\"/usr/share/ovirt-engine/bin/pki-enroll-request.sh\"\n\"--name=10.10.50.138\"\n\"--subject=/O=********/CN=10.10.50.138\"\n\"--san=IP:10.10.50.138\"\n\"--days=398\"\n\"--timeout=30\"\n\"--ca-file=ca\"\n\"--cert-dir=certs\"\n\"--req-dir=requests\"\n",
              "warn" : true,
              "_uses_shell" : false,
              "stdin_add_newline" : true,
              "strip_empty_ends" : true,
              "argv" : null,
              "chdir" : null,
              "executable" : null,
              "creates" : null,
              "removes" : null,
              "stdin" : null
            }
          },
          "stdout_lines" : [ ],
          "stderr_lines" : [ "Using configuration from openssl.conf", "unable to load number from serial.txt", "error while loading serial number", "140631516194624:error:0D066096:asn1 encoding routines:a2i_ASN1_INTEGER:short line:crypto/asn1/f_int.c:140:", "Cannot sign certificate" ],
          "_ansible_no_log" : false,
          "item" : {
...
```

发现是engine上的serial.txt文件缺失，按道理在/etc/pki/ovirt-engine上应该存在一个serial.txt文件的，这里我不知道为什么丢失了

查看这个[request](https://lists.ovirt.org/archives/list/users@ovirt.org/thread/UYEPTHOU2B6DDMVSRJNUTT7ESAE7GSSM/#AWB77BK4CLJZ34PMN45MOE4TPFK7GPLD)，有相似的情况，最后是重新创建该文件并写入1013到文件内。

```
cd /etc/pki/ovirt-engine
touch serial.txt
echo "1013" >> serial.txt
chown ovirt:ovirt serial.txt
```

## 如何进入virsh管理界面

/etc/ovirt-hosted-engine/virsh_auth.conf这个文件包含了authname和密码

```
virsh -c qemu:///system?authfile=/etc/ovirt-hosted-engine/virsh_auth.conf
```