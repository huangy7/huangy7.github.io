---
layout:     post
title:      tc 命令
subtitle:   tc 命令
date:       2020-12-16
author:     HY
header-img: img/tag-bg-o.jpg
catalog: true
tags:
    - Linux - QOS
---

# 基本步骤
1. 针对网络物理设备，绑定一个队列qdisc
2. 在该队列上建立分类class
3. 为每一个分类建立过滤器filter

# 使用
1.为网卡eth0配置一个htb队列

```bash
tc qdisc add dev eth0 root handle 1: htb default 11
# root 表示为eth0添加的是一个根队列
# handle 1: 这个根队列的句柄是1:
# htb 表示要添加的队列是htb队列
# default 是htb的特有参数，意思是所有未分类的流量都将分配给类别1:11。
```
2.为根队列创建类别

```bash
tc class add dev eth0 parent 1: classid 1:1 htb rate 40mbit ceil 40mbit  
tc class add dev eth0 parent 1: classid 1:12 htb rate 40mbit ceil 40mbit  
tc class add dev eth0 parent 1: cllassid 1:13 htb rate 20mbit ceil 20mbit
# "parent 1:", 表示类别的父亲为根队列1:
# "classid1:1", 表示创建一个标识为1:1的类别"
# "rate 40mbit", 表示系统将为该类别确保带宽40mbit
# "ceil 40mbit"，表示该类别的最高可占用带宽为40mbit。
```
3.创建过滤器

```bash
#ip限速
tc filter add dev rhpvif1 protocol ip parent 1:0 prio 1 u32 match ip dst 10.0.0.2/32 flowid 1:11
tc filter add dev rhpvif1 protocol ip parent 1:0 prio 1 u32 match ip dst 10.0.0.2/32 match ip src 10.0.0.2/32 flowid 1:11
#端口限速
tc filter add dev rhpvif1 protocol ip parent 1:0 prio 1 u32 match ip dport 80 0xffff flowid 1:11
tc filter add dev rhpvif1 protocol ip parent 1:0 prio 1 u32 match ip sport 12345 0xffff flowid 1:11
#协议限速
tc filter add dev rhpvif1 protocol ip parent 1:10 prio 1 u32 match udp match ip dst 172.16.14.209/32 flowid 1:11
tc filter add dev rhpvif1 protocol ip parent 1:10 prio 1 u32 match ip match ip protocol 1 0xff flowid 1:11
tc filter add dev rhpvif1 protocol ip parent 1:10 prio 1 u32 match ip match ip protocol 6 0xff flowid 1:11
tc filter add dev rhpvif1 protocol ip parent 1:10 prio 1 u32 match ip match ip protocol 8 0xff flowid 1:11

# protocol IP 表示这个过滤器要检查协议字段
# prio 优先级，系统由小到大
# u32选择器 来匹配不同的数据流
```

# 更高级一点的

```bash
tc qdisc add dev eth0 root handle 1: htb default 21  
tc class add dev eth0 partent 1: classid 1:1 htb rate 20mbit ceil 20mbit  
tc class add dev eth0 parent 1: classid 1:2 htb rate 80mbit ceil 80mbit  
tc class add dev eth0 parent 1:2 classid 1:21 htb rate 40mbit ceil 80mbit  
#tc class add dev eth0 parent 1:2 classid 1:22 htb rate 40mbit ceil 80mbit  
tc filter add dev eth0 protocol parent 10 prio 1 u32 match ip dport 80 0xffff flowid 1:21  
tc filter add dev eth0 protocol parent 1:0 prio 1 u32 match ip dport 25 0xffff flowid 1:22  
tc filter add dev eth0 protocol parent 1:0 prio 1 u32 match ip dport 23 0xffff flowid 1:1
# 这里为根队列1创建两个根类别，即1:1和1:2
# 在1:2中，创建两个子类别1:21和1:22
# 由于类别1:21和1:22是类别1:2的子类别，因此他们可以共享分配的80Mbit带宽。同时，又确保当需要时，自己的带宽至少有40Mbit。
```


