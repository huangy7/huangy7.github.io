---
layout:     post
title:      Linux Tunnel详解
subtitle:   Focus on Generic_Routing_Encapsulation
date:       2019-11-8
author:     HY
header-img: img/post-bg-iWatch.jpg
catalog: true
tags:
    - GRE
    - Tunnel
    - Linux
---
# 一、Tunnel 概念
所谓隧道，就是将下一层（比如 IPV4 层）的包封装到上一层（比如 SSH、HTTP）或者同一层（比如 IPV6层）的协议中进行传输，发送端和接收端都有解析这种包的程序或者内核模块。
# 二、常见的隧道协议概览

    [root@pop src]# ip tunnel help
    Usage: ip tunnel { add | change | del | show | prl | 6rd } [ NAME ]
              [ mode { ipip | gre | sit | isatap | vti } ] [ remote ADDR ] [ local ADDR ]
              [ [i|o]seq ] [ [i|o]key KEY ] [ [i|o]csum ]
              [ prl-default ADDR ] [ prl-nodefault ADDR ] [ prl-delete ADDR ]
              [ 6rd-prefix ADDR ] [ 6rd-relay_prefix ADDR ] [ 6rd-reset ]
              [ ttl TTL ] [ tos TOS ] [ [no]pmtudisc ] [ dev PHYS_DEV ]
    Where: NAME := STRING
           ADDR := { IP_ADDRESS | any }
           TOS  := { STRING | 00..ff | inherit | inherit/STRING | inherit/00..ff }
           TTL  := { 1..255 | inherit }
           KEY  := { DOTTED_QUAD | NUMBER }

### 2.1 基于数据包的隧道协议：
- ipsec
- gre -- 通用路由封装协议，支持多种网络层协议和多路技术
- ip in ip -- 比GRE更小的负载头，并且适合只有负载一个IP流的情况
- L2TP -- 数据链接层隧道协议
- MPLS -- 多协议标签交换
- GTP
- PPTP -- 点对点隧道协议
- PPPoE -- 基于以太网的点对点隧道
- PPPoA -- 基于ATM的点对点隧道
- IEEE 802.1Q -- 以太网VLANs
- DLSw -- SNA负载互联网协议
- XOT
- IPv6穿隧：6to4、6in4、Teredo
- Anything In Anything (AYIYA; e.g. IPv6 over UDP over IPv4, IPv4 over IPv6, IPv6 over TCP IPv4, [ICMP over TCP](http://www.cs.uit.no/~daniels/PingTunnel/) etc.)
- VPN -- eg:OpenVPN

### 2.2 基于流的隧道协议：
- 传输层安全
- SSH
- SOCKS
- HTTP CONNECT 命令 -- HTTP 做到 SSH 上，[TCP over TCP](http://sites.inka.de/bigred/devel/tcp-tcp.html)
- 各式的电路层级的代理服务器协议，如Microsoft Proxy Server的Winsock Redirection Protocol或WinGate Winsock Redirection Service.  

# 三、常见隧道协议
### 3.1 IP in IP (aka IPIP)
把IP数据包封装进另一个IP数据包，外层IP数据包头部包含了隧道入口点IP以及隧道出口点IP，而内部的 IP数据包 除了TTL的自然递减之外，完全不会改变。   

    ip tunnel ${interface name} mode ipip \
        local ${local endpoint address}   \
        remote ${remote endpoint address} \ 
        ttl ${time to live}  

### 3.2 Simple Internet Transition (aka SIT)  
将 IPv6数据包 封装在 IPv4数据包内，这样可以为没有 IPv6 接入的机器提供 Tunnel Broker服务  

    ip tunnel add ${interface name} mode sit \
        local ${local endpoint address}      \
        remote ${remote endpoint address}

### 3.3 IPv4 in IPv6 (aka IPIP6)

    ip -6 tunnel add ${interface name} mode ipip6 \
        local ${local endpoint address}           \
        remote ${remote endpoint address}

### 3.4 Generic Routing Encapsulation (aka GRE)
GRE规定了如何用一种网络协议去封装另一种网络协议的方法。[协议号为47](https://zh.wikipedia.org/wiki/IP%E5%8D%8F%E8%AE%AE%E5%8F%B7%E5%88%97%E8%A1%A8)。GRE的隧道由两端的源IP地址和目的IP地址来定义，允许用户使用IP包封装IP、IPX、AppleTalk包，并支持全部的路由协议（如RIP2、OSPF等）。通过GRE，用户可以利用公共IP网络连接IPX网络、AppleTalk网络，还可以使用保留地址进行网络互连，或者对公网隐藏企业网的IP地址。  

    ip tunnel add ${interface name} mode gre \
        local ${local endpoint address}      \
        remote ${remote endpoint address}    \
        key ${key value}

#### 3.4.1 GRE 通信过程
建立GRE隧道后会出现从GRE隧道出去的路由，比如是到达10.0.0.0网段，当需要发送到10.0.0.0网段的数据时，数据会到达 GRE tunnel 设备的 ndo_start_xmit()，也就是 ipgre_tunnel_xmit() 函数。这个函数干的事就是通过 tunnel 的 header_ops 构造一个新的头，即GRE头，并把相应的外部IP地址填进去，最后发送出去。如图所示：  
![Alt text](/img/articles/gre/gre1.png)  
当接收端收到这个包时，它暂时还不知道这个是 GRE 的包，它首先会把它当作普通的 IP 包处理。因为外部的 IP 头的目的地址是该路由器的地址，所以它自己会接收这个包，把它交给上层，到了 IP 层之后才发现这个包不是 TCP，UDP，而是 GRE，这时内核会转交给 GRE 模块处理。GRE 模块注册了一个 hook：  

    static const struct gre_protocol ipgre_protocol = {
            .handler = ipgre_rcv,
            .err_handler = ipgre_err,
    };

所以真正处理这个包的函数是 ipgre_rcv() 。ipgre_rcv() 所做的工作是：通过外层IP 头找到对应的 tunnel，然后剥去外层 IP 头，把这个“新的”包重新交给 IP 栈去处理，像接收到普通 IP 包一样。到了这里，“新的”包处理和其它普通的 IP 包已经没有什么两样了：根据 IP 头中目的地址转发给相应的 host。注意：这里我所谓的“新的”包其实并不新，内核用的还是同一个copy，只是skbuff 里相应的指针变了。  
#### 3.4.2 GRE 格式
![Alt text](/img/articles/gre/gre2.png)  

#### 3.4.3 遇到NAT情况
![Alt text](/img/articles/gre/gre3.png)  
在 CPE 端  

    ip tunnel add tunnel0 mode gre \
        local 10.0.0.1      \
        remote 172.16.0.2   \

在 POP 端配置的remote IP 必须是 NAT 过后的公网IP。

    ip tunnel add tunnel0 mode gre \
        local 172.16.0.2      \
        remote 172.16.0.1   \

若 POP 端 remote IP 配置成10.0.0.1，则无法建立通信
![Alt text](/img/articles/gre/gre4.png)  

#### 3.4.4 GRE 点对多点的问题（未实现）
在 CPE 端  

    ip tunnel add tunnel0 mode gre \
        local 10.0.0.1      \
        remote 172.16.0.2   \
        key 1234

在 POP 端  

    ip tunnel add tunnel0 mode gre \
        local 172.16.0.2      \
        key 1234

这样配置的隧道缺少了远端出口的地址，所以 key 是鉴别隧道流量的唯一方法，因此 key 是必须的，但是这样是 **行不通** 的。
不过在正常都配置了 local 和 remote 的时候 key 还是管用的。

#### 3.4.5 GRE keepalive
思科的GRE隧道有实现 [keepalive](https://www.cisco.com/c/zh_cn/support/docs/ip/generic-routing-encapsulation-gre/118370-technote-gre-00.html)，但是linux还没有实现。  
如果没有 keepalive 在有NAT的场景下，当NAT Session失效，即（`/proc/net/nf_conntrack`里的那条流消失），非主动发起方要是主动往NAT后端的设备发数据，会由于NAT的反向过程导致隧道不通。暂时有个办法是通过`ping`来保持住那条流，保持隧道一直畅通。可以写个`service`  

    # keepalive@.service
    [Unit]
    Description = Initializor of Private Network
    Wants=network-init.service
    After=network-init.service
    [Service]
    ExecStart=/bin/ping -i 10 -s 0 10.0.0.%i
    User=nobody
    [Install]
    WantedBy=default.target

另一种更优雅的方式：
VXLAN over gre ? 还没有实验过 



### 3.5 IP Security (aka ipsec)


 

# 三、参考文档  

- [GRE vs IPIP Tunneling](https://packetlife.net/blog/2012/feb/27/gre-vs-ipip-tunneling/)
- [深入理解GRE tunnel](http://wangcong.org/2012/11/08/-e6-b7-b1-e5-85-a5-e7-90-86-e8-a7-a3-gre-tunnel/)
- [GRE 技术介绍](http://www.h3c.com/cn/d_200805/605933_30003_0.htm)
- 
- [Linux 常见隧道协议整理及用法](https://www.soasurs.com/2018/12/18/Archives-Of-Tunnel-On-Linux-And-Usages/)

