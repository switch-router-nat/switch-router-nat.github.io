---
layout    : post
title     : "[译]理解IPv6：什么是被请求节点(solicited-node)组播(预备知识)"
date      : 2018-10-15
lastupdate: 2018-10-15
categories: Translate
---

原文地址:[understanding-ipv6-prepping-solicited-node-multicast](https://www.networkcomputing.com/networking/understanding-ipv6-prepping-solicited-node-multicast)

Solicited-node 组播: 这个名词读起来真是拗口！最初，我直接就将它略过了，没想到它却成我学习邻居发现协议(Neighbor Discovery Protocol, NDP)的障碍。

在开始 solicited-node 组播的内容之前，我们先来复习一下 link-local 组播地址。

<p align="center"><img src="/assets/img/ipv6-prepping-solicited-node-multicast/prep-1.jpg"></p>

**组播在身边无处不在**

在现在广泛运行的 IPv4 网络中，组播无处不在。如果你没有使能 IP 组播路由或者 PIM，那么你可能不这么认为。但事实就是这样，组播真的在你身边无处不在。

<p align="center"><img src="/assets/img/ipv6-prepping-solicited-node-multicast/prep-2.jpg"></p>

回到两个路由器直连的组网环境中。我们现在只使能了路由器的 IPv4 单栈功能。

**显示 IP 地址**
 
这个命令会显示许多信息，现在我们着重关注"multicast reserved groups joined" 这一行

<p align="center"><img src="/assets/img/ipv6-prepping-solicited-node-multicast/prep-3.jpg"></p>

看到了吗？如此多的组播地址！而且这些地址都属于[Internet Assigned Numbers Authority (IANA)](http://www.iana.org/assignments/multicast-addresses/multicast-addresses.xhtml#multicast-addresses-1)中描述的"Local Network Control Block (224.0.0.0 - 224.0.0.255(224.0.0/24))"

这表示接口 gig1/0/1 会监听这些组播地址对应的 MAC 地址的的报文。

<p align="center"><img src="/assets/img/ipv6-prepping-solicited-node-multicast/ipv4-addr-map.PNG"></p>

有没有似曾相识的感觉了？如前面所述，组播广泛存在于你身边的 IPv4 网络。你也许会认为这并不是"真的组播"，因为这些报文只存在于本地链路上，但这的确算是组播。 

正如我在上一篇[博文](http://www.networkcomputing.com/cloud-infrastructure/understanding-ipv6-a-sniffer-full-of-3s/a/d-id/1297812)中描述的一样，IPv6 也有类似的 link-local 组播地址"。

<p align="center"><img src="/assets/img/ipv6-prepping-solicited-node-multicast/ipv6-addr-map.PNG"></p>


**这些组播组有什么共通点吗？**

- Local: 他们只在在链路本地二层范围内有意义 
- Join: 他们并不是通过 IGMP 协议加入到一个组播组，而是通过单纯地侦听对应 IPv4 和 IPv6 的组播地址对应的 MAC 地址完成的。
- Common interest: 每个组播组都有特定的关注项。举个栗子，IPv4 中的 224.0.0.10 是  增强内部网关路由协议(EIGRP)使用的。这样规划的原因很简单:如果一个路由器希望使用 EIGRP ，那么它就侦听其对应的 MAC 地址就行了。

