---
layout:     post
title:      Linux常用命令之netstat
subtitle:   Linux
date:       2020-04-18
author:     qin4zhang
header-img: img/post-bg-universe.jpg 
catalog: true 						# 是否归档
tags:								#标签
    - Linux
    - netstat

---
# 注意
> 想法及时记录，实现可以待做。

## 介绍
netstat 用来打印Linux中网络系统的状态信息。

## 语法

```
netstat 选项

-a或--all：显示所有连线中的Socket；
-A<网络类型>或--<网络类型>：列出该网络类型连线中的相关地址；
-c或--continuous：持续列出网络状态；
-C或--cache：显示路由器配置的快取信息；
-e或--extend：显示网络其他相关信息；
-F或--fib：显示FIB；
-g或--groups：显示多重广播功能群组组员名单；
-h或--help：在线帮助；
-i或--interfaces：显示网络界面信息表单；
-l或--listening：显示监控中的服务器的Socket；
-M或--masquerade：显示伪装的网络连线；
-n或--numeric：直接使用ip地址，而不通过域名服务器；
-N或--netlink或--symbolic：显示网络硬件外围设备的符号连接名称；
-o或--timers：显示计时器；
-p或--programs：显示正在使用Socket的程序识别码和程序名称；
-r或--route：显示Routing Table；
-s或--statistice：显示网络工作信息统计表；
-t或--tcp：显示TCP传输协议的连线状况；
-u或--udp：显示UDP传输协议的连线状况；
-v或--verbose：显示指令执行过程；
-V或--version：显示版本信息；
-w或--raw：显示RAW传输协议的连线状况；
-x或--unix：此参数的效果和指定"-A unix"参数相同；
--ip或--inet：此参数的效果和指定"-A inet"参数相同。
```

## 示例

- 列出所有端口 (包括监听和未监听的)

```bash
netstat -a     #列出所有端口
netstat -at    #列出所有tcp端口
netstat -au    #列出所有udp端口  
```

- 列出所有处于监听状态的 Sockets

```bash
netstat -l        #只显示监听端口
netstat -lt       #只列出所有监听 tcp 端口
netstat -lu       #只列出所有监听 udp 端口
netstat -lx       #只列出所有监听 UNIX 端口
```

- 显示每个协议的统计信息

```bash
netstat -s   显示所有端口的统计信息
netstat -st   显示TCP端口的统计信息
netstat -su   显示UDP端口的统计信息
```

- 显示 PID 和进程名称

```bash
netstat -pt
```

- TCP各种状态统计

```bash
netstat -nt | grep -e 127.0.0.1 -e 0.0.0.0 -e ::: -v | awk '/^tcp/ {++state[$NF]} END {for(i in state) print i,"\t",state[i]}'
```

- 查找请求数前20个IP

```bash
netstat -anlp|grep 80|grep tcp|awk '{print $5}'|awk -F: '{print $1}'|sort|uniq -c|sort -nr|head -n20
```

- 查找较多TIME_WAIT连接

```bash
netstat -n|grep TIME_WAIT|awk '{print $5}'|sort|uniq -c|sort -rn|head -n20
```

- 找查较多的SYN连接

```bash
netstat -an | grep SYN | awk '{print $5}' | awk -F: '{print $1}' | sort | uniq -c | sort -nr | more
```

- 根据端口列进程

```bash
netstat -ntlp | grep -w 80 | awk '{print $7}' | cut -d/ -f1
```

- 查看java的进程数

```bash
netstat -anpo | grep "java" | wc -l
```




## 参考

1. <a href="https://man.linuxde.net/netstat" target="_blank">netstat命令</a>
2. <a href="https://www.pdai.tech/md/develop/protocol/dev-protocol-tool-netstat.html#netstat%E7%9A%84%E5%8F%82%E6%95%B0" target="_blank">netstat查看服务及监听端口详解</a>

