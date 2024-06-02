---
layout: post
title: Linux 下给 Milk-V 共享网络
date: 2024-06-02 21:23:00 +0800
tags: 
- 编程
render_with_liquid: false
---

这两天发现 Linux 下默认不会给 USB RNDIS 连接的 Milk-V 联网能力，非常难受。在 Windows 下可以使用桥接的方法简单操作（参考 [Luckfox 的联网教程](https://wiki.luckfox.com/zh/Luckfox-Pico/Luckfox-Pico-Network-Sharing-2/)），Linux 下的资料则众说纷纭。

## 链路层

如果有以太网接口，那直接连就好了，不过可能有交叉线的问题。而嵌入式里更好用的是用 USB 口，可以实现“一线连”。
Linux 中，配置了 RNDIS 的 USB 设备会被识别为网卡，在链路层和其他的实现，比如以太网或者 WiFi 没有本质的区别，都可以同样使用。而主要的难点在于在从设备中启用 RNDIS。不过好在大部分没有以太网接口的设备已经默认启用了该功能，不用担心。

官方的文档已经提供了配置的方法，链路层配好后，两个设备可以用 IP 地址互相访问，比如 SSH 就基于此连接。另外也可以用`nc -l -p 1234`开启一个监听端口，另一端用`echo hello | nc 192.168.12.34 1234`测试。

## 网络层

网络层是比较棘手的问题，这个问题分为两方面：

- 开发板一端需要配置网关，将 IP 包发送到主机。
- 主机需要处理上述 IP 包。

### 查看地址

默认应该只有一个 USB 网卡，即`usb0`，可以用`ip address show dev usb0`查看本机地址。

```shell
melonedo@linux:~$ ip a s dev usb0 # ip address show dev usb0
10: usb0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UNKNOWN group default qlen 1000
    link/ether ba:dd:53:11:a1:b6 brd ff:ff:ff:ff:ff:ff
    inet 192.168.42.98/24 brd 192.168.42.255 scope global dynamic noprefixroute usb0
       valid_lft 650sec preferred_lft 650sec
    inet6 fe80::6c5e:8610:7749:7596/64 scope link noprefixroute
       valid_lft forever preferred_lft forever
```

这里的 192.168.42.98 就是主机的 IP。或者在开发板运行

```shell
[root@milkv-duo]~# arp
? (192.168.42.98) at ba:dd:53:11:a1:b6 [ether]  on usb0
```

可以找到连接设备另一端的地址。


### 开发板配置网关

配置前，访问任意的地址，会立刻报错。

```shell
[root@milkv-duo]~# ping 223.5.5.5
PING 223.5.5.5 (223.5.5.5): 56 data bytes
ping: sendto: Network unreachable
```

这是因为 Milk-V 开发板开启了 DHCP 服务器，默认没有配置网关。实际上这里的默认网关是他自己，但 Milk-V 没有联网的能力。我们手动将主机的地址设为网关。

```shell
route add default gw 192.168.42.98 usb0
```

这样，再次访问，可以发现不再立刻报错而是一直超时：

```shell
[root@milkv-duo]~# ping 223.5.5.5
PING 223.5.5.5 (223.5.5.5): 56 data bytes
^C
--- 223.5.5.5 ping statistics ---
2 packets transmitted, 0 packets received, 100% packet loss
```

在主机端用`sudo tcpdump -i usb0`检查，会发现 ICMP 包不断发出，但没有回复，说明问题出在主机.

```shell
melonedo@linux:~$ sudo tcpdump -i usb0
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on usb0, link-type EN10MB (Ethernet), capture size 262144 bytes
22:19:57.641850 IP 192.168.42.1 > 223.5.5.5: ICMP echo request, id 770, seq 0, length 64
22:19:58.642178 IP 192.168.42.1 > 223.5.5.5: ICMP echo request, id 770, seq 1, length 64
22:19:59.642635 IP 192.168.42.1 > 223.5.5.5: ICMP echo request, id 770, seq 2, length 64
22:20:00.643064 IP 192.168.42.1 > 223.5.5.5: ICMP echo request, id 770, seq 3, length 64
```

### 主机配置转发

在主机端设置转发主要参考[How to Use Linux as a Gateway](https://www.baeldung.com/linux/network-gateway)，这里简单总结一下（这里的操作都不会在重启后保存）：

首先启用端口转发

```shell
sudo sysctl -w net.ipv4.ip_forward=1
```

然后设置 MASQUERADE，这里 usb0 是 RNDIS 网卡，而以太网接口为 enp0s31f6：

```shell
sudo iptables -t nat -A POSTROUTING -o enp0s31f6 -j MASQUERADE
sudo iptables -A FORWARD -i enp0s31f6 -o usb0 -m state --state RELATED,ESTABLISHED -j ACCEPT
sudo iptables -A FORWARD -i usb0 -o enp0s31f6 -j ACCEPT
```

设置后可以用`sudo iptables --list-rules`查看。

这样设置后，再`ping 223.5.5.5`，连接成功。

```shell
[root@milkv-duo]~# ping 223.5.5.5
PING 223.5.5.5 (223.5.5.5): 56 data bytes
64 bytes from 223.5.5.5: seq=0 ttl=111 time=5.680 ms
64 bytes from 223.5.5.5: seq=1 ttl=111 time=5.381 ms
```

## 应用层

最后，要想访问域名，还需要配置 DNS。这步比较简单，直接在 /etc/resolv.conf 里添加`nameserver 223.5.5.5`即可。

