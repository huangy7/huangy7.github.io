---
layout:     post
title:      如何开启 windows 路由转发功能
subtitle:   win10 IP Routing
date:       2019-10-31
author:     HY
header-img: img/post-bg-debug.png
catalog: true
tags:
    - windows
    - win 10
    - IP FORWARDING
---

# 开启 win10 转发功能
以管理员身份运行 CMD
![Alt text](/img/articles/ip_forward_win/1161141.png)

将 `HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\Tcpip\Parameters\IPEnableRoute`设为1
```powershell
reg add HKLM\SYSTEM\CurrentControlSet\Services\Tcpip\Parameters /v IPEnableRouter /D 1 /f
```
![Alt text](/img/articles/ip_forward_win/1161210.png)

将 `Routing and Remote Access` 服务的启动类型更改为自动并启动服务
```powershell
sc config RemoteAccess start= auto
sc start RemoteAccess
```
![Alt text](/img/articles/ip_forward_win/1161240.png)

# 参考文档
- [windows-howto-enable-ip-routing](https://michlstechblog.info/blog/windows-howto-enable-ip-routing/) 
