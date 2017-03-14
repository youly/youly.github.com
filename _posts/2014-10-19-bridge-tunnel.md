---
layout: post
title: 网络基础-bridge和tunnel
category: 协议
tags: [network, tunnel, bridge]
---

### 什么是bridge

#### 概念
桥，字面上可以理解为连接两个不同网络段的“桥梁”，但实际上最开始是为了划分网络段，隔离冲突域。

以太网中数据包以广播的形式发送，每台设备收到数据包后需要判断这个包是否是发给自己的，如果目标mac地址与自己的相同，则接受数据包、然后流向网络层，否则丢弃。由于所有设备共享网络传输介质，导致了如果多台设备同时发送数据包则会引发冲突。一旦设备过多，整个网络传输效率就变得低下。为了减少冲突的发生，出现了桥接器，将一个大的网络分离成两个小的网络，数据包到达桥接器时会以mac地址来判断是否要继续广播。这里桥接器内部维护一个mac地址与端口的映射表，通过多次的数据广播学习而得。

桥接器工作于OSI模型中的链路层，交换机可以理解为有多个的端口的桥接器，连接多个不同的网段。

#### 应用
创建一个需要上网的虚拟机，而系统只有一个网卡，这时候虚拟交换机就派上用场了。linux里自带了一个bridge内核模块，设置可参考：[https://wiki.archlinux.org/index.php/Network_bridge](https://wiki.archlinux.org/index.php/Network_bridge)

### 什么是tunnel

#### 概念
[隧道](http://en.wikipedia.org/wiki/Tunneling_protocol)，在原有的传输层协议上对数据包的自定义协议封装。比较广泛的是ip tunnel，在ip（公网地址）协议之上传输ip（私有地址）数据包。

#### 应用
ssh端口转发、openvpn

### 参考
1、[在freebsd和linux之间创建ip-ip tuannel](http://kovyrin.net/2006/03/17/how-to-create-ip-ip-tunnel-between-freebsd-and-linux/)






