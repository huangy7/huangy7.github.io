---
layout:     post
title:      Linux Bridge 详解
subtitle:   网桥指令 功能等 
date:       2019-11-1
author:     HY
header-img: img/post-bg-debug.png
catalog: true
tags:
    - Bridge
    - Liunx
    - VLAN
---
# 一、vlan_filter 功能
内核3.8版本以后的 bridge 增加了一个新功能叫 **VLAN filter**  
**VLAN filter** 主要有两个功能
1. 可以给进出的数据打vlan tag ，剥离vlan tag，要看后面跟的参数
2. 起到过滤功能，比如接口vid设置为100,则只让vlan100的数据通过，直接丢弃其他数据  

废话不多说，直接上命令

    $ ip link set br0 type bridge vlan_filtering 1
    或者
    $ echo 1 > /sys/class/net/brx/bridge/vlan_filtering
    $ bridge vlan add dev veth01 vid 100 pvid untagged master

`bridge vlan add` -- 增加一个VLAN过滤条目
- `dev NAME` -- 与此VLAN相关的接口
- `vid VID` -- 标识VLAN的VLAN ID
- `tunnel_info TUNNEL_ID` -- 映射到此vlan的隧道ID(没用过，不懂什么意思)
- `pvid` -- 指定的VLAN在入口时将被视为PVID。任何未标记的帧都将分配给该VLAN
- `untagged` -- 指定的VLAN在出口时将被视为未标记
- `self` -- 在指定的物理设备上配置了VLAN。如果该设备是bridge，则必选
- `master` -- 默认配置  
`bridge vlan delete`
`bridge vlan show`
`bridge vlan tunnelshow`  

有了 VLAN filter, Linux Bridge 就像一个真正的交换机，我们不再需要创建多个 VLAN 和网桥，直接在 Bridge 上打上/剥离 VLAN 标记。

# 二、SVI 接口 -- VLAN 间路由
一个交换机可能会有多个VLAN与其相连，则需要通过三层流量让这些VLAN间进行通信。而SVI(Switch virtual interface)即为路由间选择的虚拟VLAN接口。
![](/img/articles/4104220.png)  
废话不多说直接上命令  
增加SVI接口  

    $ ip link add link br0 name br0.100 type vlan id 100
    $ ip addr add 192.168.1.1/24 dev br0.100

连接VM1和VM2的接口增加VLAN过滤ID 100 和 200  

    $ ip link set br0 type bridge vlan_filtering 1
    $ bridge vlan add dev veth1 vid 100 pvid untagged master


VM1和VM2的默认网关指向SVI接口  

    $ ip route rep default via 192.168.1.1

**关键一步：br0也要增加VLAN过滤ID 100 和 200**。当VM1第一次想要发数据到VM2，会先问网关的MAC地址，ARP请求到达veth1后会打上ID=100的VLAN TAG。然后数据会到达br0.100，br0.100会作ARP应答，数据到达br0后，若br0没有这增加这两个ID的VLAN过滤条目，则会将数据丢弃。所以 **br0也要增加VLAN过滤ID 100 和 200**（感觉解释得不太清楚，反正就要打，不打数据就会被丢 :sweat_smile:）  

    $ bridge vlan add dev br0 vid 100 pvid untagged self

>其实要不要加这个 `pvid` `untagged`我也没太搞清楚……:sweat:评论区有大神指点就好了。

# 三、其他功能
基本操作，废话不多说。

# 四、参考文档
- [vlan-filter-support-on-bridge](https://developers.redhat.com/blog/2017/09/14/vlan-filter-support-on-bridge/)
- [bridge 命令手册](http://man7.org/linux/man-pages/man8/bridge.8.html)
- [Openstack Neutron：二层技术和实现](https://www.cnblogs.com/pmyewei/p/6413875.html/)
- [PVID和VID的理解](https://blog.csdn.net/silent123go/article/details/70146806)
- [实施VLAN间路由](http://icon.clnchina.com.cn/pdf/CCNPSWITCH-xuexizhinan-04.pdf)

