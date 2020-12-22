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

Linux中的QoS分为入口(Ingress)部分和出口(Egress)部分，入口部分主要用于进行入口流量限速(policing)，出口部分主要用于队列调度(queuing scheduling)。大多数排队规则(qdisc)都是用于输出方向的，输入方向只有一个排队规则，即ingress qdisc

# TC限出方向
## 基本步骤
1. 针对网络物理设备，绑定一个队列qdisc
2. 在该队列上建立分类class
3. 为每一个分类建立过滤器filter

## 使用
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

3.为每一个类设置sfq，使用随机公平队列，保证不会有某一个回话独占出口带宽
```bash
#为了避免一个会话永占带宽,添加随即公平队列sfq.
tc qdisc add dev eth0 parent 1:12 handle 12: sfq perturb 10
#perturb：是多少秒后重新配置一次散列算法，默认为10秒
#sfq,他可以防止一个段内的一个ip占用整个带宽
```

4.创建过滤器

```bash
#ip限速，使用u32创建过滤器
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
#iptables打mark限速
tc filter add dev swtun1 protocol ip parent1:0 prio 1 handle 1 fw classid 1:11
#高优先级流量打mark
iptables -t mangle -A POSTROUTING -p tcp -m tcp --sport 9998 -o swtun1 -j MARK --set-mark 1
# protocol IP 表示这个过滤器要检查协议字段
# prio 优先级，系统由小到大
# u32选择器 来匹配不同的数据流
```

## 更高级一点的

```bash
tc qdisc add dev eth0 root handle 1: htb default 21  
tc class add dev eth0 partent 1: classid 1:1 htb rate 20mbit ceil 20mbit  
tc class add dev eth0 parent 1: classid 1:2 htb rate 80mbit ceil 80mbit  
tc class add dev eth0 parent 1:2 classid 1:21 htb rate 40mbit ceil 80mbit  
tc class add dev eth0 parent 1:2 classid 1:22 htb rate 40mbit ceil 80mbit  
tc filter add dev eth0 protocol parent 10 prio 1 u32 match ip dport 80 0xffff flowid 1:21  
tc filter add dev eth0 protocol parent 1:0 prio 1 u32 match ip dport 25 0xffff flowid 1:22  
tc filter add dev eth0 protocol parent 1:0 prio 1 u32 match ip dport 23 0xffff flowid 1:1
# 这里为根队列1创建两个根类别，即1:1和1:2
# 在1:2中，创建两个子类别1:21和1:22
# 由于类别1:21和1:22是类别1:2的子类别，因此他们可以共享分配的80Mbit带宽。同时，又确保当需要时，自己的带宽至少有40Mbit。
```

# tc 限入方向
根据tc相关文档描述，使用tc ingress限速，功能有限，似乎只能选择丢弃，并且也不支持分类。实际应用中，我们可以将业务流重定向到ifb设备上，业务流从这个ifb设备中出去，再由相应的网口接收，那我们就可以像正常使用tc对egress限速一样，来对ifb设备进行egress限速，就可以达到对接收方向的限速了。
## 基本步骤
1.启动ifb模块，添加ifb网卡
2.将真实网卡的出流量重定向到ifb网卡
3.配置ifb网卡的过滤规则

ifb驱动模拟一块虚拟网卡，它可以被看作是一个只有TC过滤功能的虚拟网卡，说它只有过滤功能，是因为它并不改变数据包的方向，即对于往外发的数据包被重定向到ifb之后，经过ifb的TC过滤之后，依然是通过重定向之前的网卡发出去，对于一个网卡接收的数据包，被重定向到ifb之后，经过ifb的TC过滤之后，依然被重定向之前的网卡继续进行接收处理，不管是从一块网卡发送数据包还是从一块网卡接收数据包，重定向到ifb之后，都要经过一个经由ifb虚拟网卡的dev_queue_xmit操作。说了这么多，看个图就明白了
![](/img/articles/tc/ifb.jpeg)

## 使用
```bash
# ifb模块需要手动加载。
modprobe ifb

# 启用虚拟设备ifb0
ip link set dev ifb0 up

#重定向ens3f3入口流量到ifb0
tc qdisc add dev ens3f3 handle ffff: ingress
tc filter add dev ens3f3 parent ffff: protocol ip u32 match u32 0 0 action mirred egress redirect dev ifb0

# 配置ifb网卡过滤规则
tc qdisc add dev ifb0 root handle 1: htb default 20
tc class add dev ifb0 parent 1: classid 1:1 htb rate 10000mbit
tc filter add dev ifb0 protocol ip parent 1:0 prio 1 u32 match ip src 129.9.123.85 flowid 1:1
```