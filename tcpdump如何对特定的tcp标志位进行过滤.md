---
title: tcpdump如何对特定的tcp标志位进行过滤
description: 
published: true
date: 2022-07-09T08:12:58.908Z
tags: tcpdump, tcp
editor: markdown
dateCreated: 2022-06-23T12:33:22.176Z
---

根据tcp保文结构可知，TCP标志头位于头的第14字节中，因为编号从0字节开始，所以TCP标志头在第13字节。

![tcp_header.png](https://ucc.alicdn.com/pic/developer-ecology/e48446f2da8946e3a7a818782663dc37.png "tcp_header.png")

字节13最多可以包含8个单比特标志；但是，TCP只能使用6个标志。其他两个位是保留的，应该设置为零。

对于只有一个标志的TCP头，每一位都有一个字节，字节13包含以下十进制的二进制值。

-   Final (FIN) = 1  
    
-   Sync (SYN) = 2  
    
-   Reset (RST) = 4  
    
-   Push (PSH) = 8  
    
-   Acknowledgement (ACK) = 16  
    
-   Urgent (URG) = 32  
    
-   Reserved = 64 and 128  
    

如果为TCP头设置了多个标志，字节13的值是所有被设置的位的二进制值之和。例如

-   FIN, ACK = 17 (1 + 16)  
    
-   SYN, ACK = 18 (2 + 16)  
    
-   PSH, ACK = 24 (8 + 16)  
    
-   FIN, PSH = 9 (1 + 8)  
    
-   FIN, PSH, ACK = 25 (1 + 8 + 16)  
    

用过滤SYN举例

```
[root@ucloud ~]
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), capture size 262144 bytes
23:28:45.929246 IP 113.65.30.42.jomamqmonitor > 10.13.132.171.ssh: Flags [S], seq 356758948, win 64240, options [mss 1412,nop,wscale 8,nop,nop,sackOK], length 0
23:28:55.109148 IP 113.65.30.42.netscript > 10.13.132.171.ssh: Flags [S], seq 1672259268, win 64240, options [mss 1412,nop,wscale 8,nop,nop,sackOK], length 0
23:29:06.584163 IP 128.199.4.167.45848 > 10.13.132.171.ssh: Flags [S], seq 572498397, win 42340, options [mss 1412,sackOK,TS val 2388703754 ecr 0,nop,wscale 8], length 0
```

假如需要过滤SYN+ACK的包，则是SYN, ACK = 18 (2 + 16)。像这样

```
[root@ucloud ~]
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), capture size 262144 bytes
23:30:41.332866 IP 10.13.132.171.ssh > 113.65.30.42.macbak: Flags [S.], seq 1578017406, ack 2299936850, win 64952, options [mss 1412,nop,nop,sackOK,nop,wscale 8], length 0
23:30:43.381328 IP 10.13.132.171.ssh > 188.166.240.30.34898: Flags [S.], seq 3237261487, ack 715567311, win 64400, options [mss 1412,sackOK,TS val 498768258 ecr 1839524773,nop,wscale 8], length 0
23:30:45.616443 IP 10.13.132.171.ssh > 113.65.30.42.wcpp: Flags [S.], seq 3406886282, ack 1487533276, win 64952, options [mss 1412,nop,nop,sackOK,nop,wscale 8], length 0
```