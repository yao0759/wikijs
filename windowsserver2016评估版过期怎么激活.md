---
title: windows server 2016评估版过期怎么激活
description: 
published: true
date: 2022-07-09T08:13:09.782Z
tags: windows
editor: markdown
dateCreated: 2022-06-23T16:13:09.692Z
---

## windows server 2016评估版过期怎么激活

之前有个服务装在了Windows server2016评估版上，最近过期了，但是使用密钥激活提示下述信息错误 

![image-20220623235029255.png](https://ucc.alicdn.com/pic/developer-ecology/2e1e7e17128a43f587e820c5cc4bbc56.png "image-20220623235029255.png") 

尝试使用slmgr /ipk xxxxx-xxxxx-xxxxx-xxxxx-xxxxx激活，还是收到如下提示

![slmgr-Error-0xC004F069.jpg](https://ucc.alicdn.com/pic/developer-ecology/751fb80304224ef389962e85fc139e17.jpg "slmgr-Error-0xC004F069.jpg")

这种情况下需要先确认当前的版本，执行`DISM /online /Get-TargetEditions`获取当前的版本信息

![current-edition-ServerStandardEval.jpg](https://ucc.alicdn.com/pic/developer-ecology/0878ad37a23147ebb1c0a5929593369e.jpg "current-edition-ServerStandardEval.jpg")

确认是评估版的话，这种时候就要将版本升级，可以用`DISM /online /Get-TargetEditions`查询可以升级的版本

![target-edition-ServerStandard.jpg](https://ucc.alicdn.com/pic/developer-ecology/53ac889d7b37427bb2ce91b9181de98f.jpg "target-edition-ServerStandard.jpg")

在升级前需要确认几个问题

-   你不能升级一个具有活动目录域服务域控制器角色的服务器。它将首先被降级为成员服务器（检查FSMO AD角色是否在该DC上运行，如果有必要，将它们转移到其他域控制器上）。  
    
-   如果在服务器上配置了NIC Teaming，在升级前必须禁用它。  
    
-   Windows Server Eval Datacenter不能被升级到Windows Server Standard Full。首先，你需要将你的版本升级到Windows Server Datacenter Full，然后使用一个小技巧来降级Windows Server版本（查看文章末尾的链接）。  
    
-   你可以同时转换Windows Server的完整GUI版本和Windows Server Core（从Windows Server 2016 14393.0.161119-1705.RS1\_REFRESH的发布开始支持转换Server Core的试用版本）。  
    

·准备将评估版升级到许可版本，需要用到[Windows server 2016公共KMS密钥](http://woshub.com/activating-windows-server-2016-with-kms-server/#h2_5)，在cmd中执行如下命令

`dism /online /set-edition:ServerStandard /productkey:WC2BQ-8NRM3-FDDYY-2BFGV-KHKQY /accepteula`

升级完成后，就可以用密钥开始激活。