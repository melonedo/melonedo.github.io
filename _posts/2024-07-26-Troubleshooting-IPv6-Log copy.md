---
layout: post
title: IPv6 网络调试记录
date: 2024-07-26 00:02:00 +0800
tags: 
- 编程
render_with_liquid: false
---

IPv6 基本知识：
- [IPv6地址格式、邻居发现NDP、DHCPv6、SLAAC、Path-MTU（PMTU）](https://www.jianshu.com/p/3bd05c37d3b0)

- 链接本地地址：EUI-64 使用的是 fe80::200:5eff:fe00:106%wlan0 对应 MAC 为 00:00:5e:00:01:06。需要使用 %dev 说明使用的接口，其中 dev 可以是数字，顺序按照 `ip a` 的顺序。现在的 IPv6 一般用 SLAAC，不直接把 MAC 明文写在里面了。
- 全局地址：
- 组播地址：

# 链路层

```
ping fe80::200:5eff:fe00:106%wlan0
```

检查邻居表：

```
ip neigh # 缩写为 ip n
```

其中`ip`命令使用`ip -6`指定 IPv6，使用`ip -4`指定 IPv4，如`ip -6 n`，下同。

调试：
```
sudo tcpdump -i wlan0 ip6 -en
```
ip6 指定协议，-i 指定接口，-e 显示 MAC，-n 不使用 DNS（很多 IP 会转换成莫名其妙的域名，没有意义）

过滤规则

```
# sudo tcpdump -i enp0s31f6 -en 'ip6 and not (host 240e:351:6352:2f00:7756:5efd:eec:af32 or host 240e:351:6352:2f00:1d0c:da6f:9636:4dc or ether host 44:f7:70:26:ad:ec)'
sudo tcpdump -en -i wlan0 icmp6 -vvv
```

```
20:46:17.549723 00:00:5e:00:01:06 > dc:a6:32:ae:45:f5, ethertype IPv6 (0x86dd), length 86: (class 0xc0, hlim 255, next-header ICMPv6 (58) payload length: 32) fe80::200:5eff:fe00:106 > fe80::2b0e:f84d:4208:62f0: [icmp6 sum ok] ICMP6, neighbor advertisement, length 32, tgt is fe80::200:5eff:fe00:106, Flags [router, solicited, override]
          destination link-address option (2), length 8 (1): 00:00:5e:00:01:05
            0x0000:  0000 5e00 0105
```

这里 host 是 dst 或者 src 之一的意思，前缀 ether 表示后面是 MAC 而不是 IP。规则编写参考：https://www.tcpdump.org/manpages/pcap-filter.7.html

多播MAC参考：https://en.wikipedia.org/wiki/Multicast_address#Ethernet，https://en.wikipedia.org/wiki/MAC_address
MAC 查询：https://macaddress.io/

修改邻居表：
```
sudo ip neigh change fe80::200:5eff:fe00:106 dev wlan0 lladdr 00:00:5e:00:01:06 router
```

# 网络层

检查路由表：

```
ip route # 缩写为 ip r
ip -6 route # 默认只显示 IPv4 的部分
```

检查某个 IP 的路由

```
ip route get 2408:8720:806:300:70::77
2408:8720:806:300:70::77 from :: via fe80::200:5eff:fe00:106 dev wlan0 proto static src 2001:da8:8002:31f4::3:f1bd metric 600 pref medium
```

需要注意的是这一步包括几个部分，每个部分都可能出错。

- 目标 IP：查询的原因
- 源 IP：用于返回结果，也可能出错
- 目标 MAC：由于存在交换机，即使是以太网，同一个接口内也可能有多个，网卡内部用于区分各个设备的地址
- 源 MAC /接口：使用的网卡接口，同时也自然指定了所使用的 MAC

https://www.kc8apf.net/2019/03/renewing-dhcp-while-keeping-a-networkmanager-connection-up/
https://networkmanager.pages.freedesktop.org/NetworkManager/NetworkManager/nm-settings-nmcli.html
