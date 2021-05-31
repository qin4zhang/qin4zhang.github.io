---
layout:     post
title:      Linux之tcpdump介绍
subtitle:   tcpdump抓包
date:       2020-05-09
author:     qin4zhang
header-img: img/post-bg-universe.jpg 
catalog: true 						# 是否归档
tags:								#标签
    - Linux
    - tcpdump

---
# 注意
> 想法及时记录，实现可以待做。

## 简介
tcpdump是一个运行在命令行下的数据包分析器。它允许用户拦截和显示发送或收到过网络连接到该计算机的TCP/IP和其他数据包。tcpdump 是一个在BSD许可证下发布的自由软件。

tcpdump 适用于大多数的类Unix系统 操作系统：包括Linux、Solaris、BSD、Mac OS X、HP-UX和AIX 等等。在这些系统中，tcpdump 需要使用libpcap这个捕捉数据的库。其在Windows下的版本称为WinDump；它需要WinPcap驱动，相当于在Linux平台下的libpcap.

以上内容来自<a href="https://zh.wikipedia.org/wiki/Tcpdump" target="_blank">维基百科</a>

## 分析
技术开发中，经常会遇到与网络有关的问题，碰到这类问题，多半是需要我们去分析网络请求，分析网络是否与预期一致，这个时候，能抓包看下网络请求是最有效的方式。

### 安装

```

apt install tcpdump (Ubuntu)

yum install tcpdump (Redhat/Centos)

```

### 用法

```
tcpdump [options]

tcpdump -i eth0 -nn -s0 -v port 80

```

此处列举常用的参数，其他参数可以参阅文档熟悉使用。

- -n 表示不要解析域名，直接显示 ip
- -nn 不要解析域名和端口
- -s 报文长度，
- -X 同时用 hex 和 ascii 显示报文的内容
- -XX 同 -X， 但同时显示以太网头部
- -S 显示绝对的序列号（sequence number），而不是相对编号
- -i any 监听所有的网卡
- -v,-vv,-vvv 显示更多的详细信息
- -c number 截取 number 个报文，然后结束
- -w file.pcap 输出为文件

### 过滤器
网络数据是非常多的，有时候我们只关注与问题有关的数据，那么此时就会想到进行数据过滤，过滤器就是这个作用。

- Host过滤器

发往主机和从主机发出的网络数据，都会被抓取

```
tcpdump host 1.2.3.4
```

如果只想看从主机发出的网络数据，那么需要指明

```
tcpdump src host 1.2.3.4
```

如果只想看发往主机的网络数据，那么需要指明

```
tcpdump dst host 1.2.3.4
```

- Network过滤器

Network 过滤器用来过滤某个网段的数据，使用的是<a href="https://zh.wikipedia.org/wiki/Tcpdump" target="_blank">CIDR</a>模式。

可以使用四元组（x.x.x.x）、三元组（x.x.x）、二元组（x.x）和一元组（x）。四元组就是指定某个主机，三元组表示子网掩码为 255.255.255.0，二元组表示子网掩码为 255.255.0.0，一元组表示子网掩码为 255.0.0.0。

抓取所有发往网段 192.168.1.x 或从网段 192.168.1.x 发出的流量

```
tcpdump -i eth0 -s0 net 192.168.1
```

抓取所有发往网段 10.x.x.x 或从网段 10.x.x.x 发出的流量：

```
tcpdump -i eth0 -s0 net 10
```

抓取从网段 10.x.x.x 发出的流量

```
tcpdump -i eth0 -s0 src net 10
```

使用 CIDR 格式的过滤

```
tcpdump -i eth0 -s0 net 172.16.0.0/12
```

- Proto过滤器

Proto 过滤器用来过滤某个协议的数据，关键字为 proto，可省略。proto 后面可以跟上协议号或协议名称，支持 icmp, igmp, igrp, pim, ah, esp, carp, vrrp, udp和 tcp。

抓取icmp协议的报文数据

```
tcpdump -n icmp
```

- Port过滤器

Port 过滤器用来过滤通过某个端口的数据报文，关键字为 port。

```
tcpdump port 80
```

```
tcpdump portrange 21-23
```

### 示例

- 抓取特定协议的数据

```
tcpdump -i eth0 udp
tcpdump -i eth0 proto 17
tcpdump -i eth0 tcp
tcpdump -i eth0 proto 6
```

- 抓取特定主机的数据

```
tcpdump -i eth0 host 10.10.1.1
tcpdump -i eth0 src 10.10.1.1
tcpdump -i eth0 dst 10.10.1.1
```

- 将抓取的数据写入文件

这种文件可以使用图形工具，比如著名的wireshark进行分析。

```
tcpdump -i eth0 -s0 -w test.pcap
```

- 行缓冲模式

将抓取到的数据，通过管道方式做其他的处理

```
tcpdump -i eth0 -s0 -l port 80 | grep "Server:"
```

- 提取HTTP用户代理

```
tcpdump -nn -A -s1500 -l | grep "User-Agent:"

tcpdump -nn -A -s1500 -l | egrep -i 'User-Agent:|Host:'
```

- 抓取HTTP GET 和 POST 流量

抓取 HTTP GET 流量：

```
tcpdump -s 0 -A -vv 'tcp[((tcp[12:1] & 0xf0) >> 2):4] = 0x47455420'
```

抓取 HTTP POST 流量：

```
tcpdump -s 0 -A -vv 'tcp[((tcp[12:1] & 0xf0) >> 2):4] = 0x504f5354'
```

- 抓取 HTTP 请求的 URL

```
tcpdump -s 0 -v -n -l | egrep -i "POST /|GET /|Host:"
```

- 抓取 HTTP POST 请求中的密码

```
tcpdump -s 0 -A -n -l | egrep -i "POST /|pwd=|passwd=|password=|Host:"
```

- 抓取 Cookies

```
tcpdump -nn -A -s0 -l | egrep -i 'Set-Cookie|Host:|Cookie:'
```

- 抓取 ICMP 的数据包

```
tcpdump -n icmp
```

- 抓取 SMTP/POP3 协议的邮件

```
tcpdump -nn -l port 25 | grep -i 'MAIL FROM\|RCPT TO'
```

- 抓取 NTP 服务的查询和响应

```
tcpdump dst port 123
```

- 抓取 IPv6 流量

```
tcpdump -nn ip6
```

- 检测端口扫描

```
tcpdump -nn
```

- 抓取 DNS 请求和响应

```
tcpdump -i any -s0 port 53
```

- 抓取 HTTP 有效数据包
抓取 80 端口的 HTTP 有效数据包，排除 TCP 连接建立过程的数据包（SYN / FIN / ACK）

```
tcpdump 'tcp port 80 and (((ip[2:2] - ((ip[0]&0xf)<<2)) - ((tcp[12]&0xf0)>>2)) != 0)'
```

- 找出发包最多的 IP

```
tcpdump -nnn -t -c 200 | cut -f 1,2,3,4 -d '.' | sort | uniq -c | sort -nr | head -n 20
```


## 参考

1. <a href="https://zh.wikipedia.org/wiki/Tcpdump" target="_blank">维基百科</a>
2. <a href="https://cizixs.com/2015/03/12/tcpdump-introduction/" target="_blank">抓包神器 tcpdump 使用介绍</a>
3. <a href="https://juejin.cn/post/6844904084168769549" target="_blank">超详细的网络抓包神器 tcpdump 使用指南</a>
4. <a href="https://danielmiessler.com/study/tcpdump/" target="_blank">A tcpdump Tutorial with Examples — 50 Ways to Isolate Traffic</a>









