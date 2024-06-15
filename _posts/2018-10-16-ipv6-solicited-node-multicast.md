---
layout    : post
title     : "[译]理解IPv6：什么是被请求节点(solicited-node)组播"
date      : 2018-10-16
lastupdate: 2018-10-16
categories: Translate
---

原文地址:[understanding-ipv6-what-solicited-node-multicast](https://www.networkcomputing.com/networking/understanding-ipv6-what-solicited-node-multicast)

 
IPv6 solicited-node 组播---这个名词听上去的确会让初学者心生疑惑。我认为这完全是因为这个名词太陌生的缘故。因此，本文将按照 [RFC 4291](http://datatracker.ietf.org/doc/rfc4291/) 的内容详细阐述 IPv6 solicited-node 组播是什么，以及这个组播地址的创建规则是什么。

我们先来看看 solicited-node 在 IPv4 与 IPv6 中的共通点.

在这篇[博文](http://www.networkcomputing.com/networking/understanding-ipv6-prepping-for-solicited-node-multicast/a/d-id/1315730)中，我们学习了 IPv6 的本地链路范围内( link-local scope)的组播地址，并且举了一个设备使用 EIGRP 协议的的栗子，它对应的组播地址是 FF02::A。 

我们还是以 FF02::A 为例, 看看[博文](http://www.networkcomputing.com/networking/understanding-ipv6-prepping-for-solicited-node-multicast/a/d-id/1315730)中描述的组播组的 3 个共通点。

- Local: FF02::A 只在链路本地二层范围内有效；
- Join: 设备通过侦听 FF02::A 组播地址的方法加入组播组，对链路层来说，就是侦听对应的组播目的 MAC 地址，对 IPv6 来说，就是 33:33:00:00:00:0A；
- Common interest: 侦听相同组播地址的设备关注相同的协议。对于 FF02::A，侦听该地址，说明这些设备希望加入 EIGRP.

而本文的主角 Solicited-node 组播也具有上面的共通点:

- Solicited-node 组播地址同样只在链路本地二层范围内有效
- 每个加入 Solicited-node 组播组的设备都会侦听 Solicited-node 的组播地址，链路层上侦听对应的 MAC 地址;
- 所有侦听同一个 Solicited-node 组播地址的设备关心同样目的地址的报文。

<p align="center"><img src="/assets/img/ipv6-solicited-node-multicast/solicited-node-pyramid.png"></p>

接下来来看看 [RFC 4291](http://datatracker.ietf.org/doc/rfc4291/) 标准是怎么写的:

> Solicited-node 组播地址由节点的单播地址和任播地址计算而来。它的低 24 位为单播或任播地址的低 24 位，高 104 位为 FF02:0:0:0:0:1:FF00::/104。由此我们可以得到 Solicited-node 组播地址的范围为 FF02:0:0:0:0:1:FF00:0000 到 FF02:0:0:0:0:1:FFFF:FFFF.
举个栗子，假设一个节点具有一个 IPv6 地址 4037::01:800:200E:8C6C，则它对应的 solicited-node 组播地址为 FF02::1:FF0E:8C6C。所以如果两个 IPv6 地址只是 104 位的前缀不同，则它们的 solicited-node 组播地址是相同的，这种机制限制了加入一个组播组中节点的最大数量。
当在一个节点上的接口上配置单播或者任播地址(自动或手动)时，它会自动计算对应的 solicited-node 组播地址，然后加入这个组播组。

举两个栗子： 

<p align="center"><img src="/assets/img/ipv6-solicited-node-multicast/routerA.png"></p>

我们在路由器 A 的 gig1/0/1 上配置了一个 IPv6 link-local 地址 FE80::1。则他计算出来的 solicited-node 组播地址为 FF02::1::FF00:1

<p align="center"><img src="/assets/img/ipv6-solicited-node-multicast/join-text.png"></p>

如果接口上配置多个地址呢？如下图所示, gig1/0/1 除了 FE80::1 外，又配置了一个全球单播地址 2001:DB8::AB:1/64

<p align="center"><img src="/assets/img/ipv6-solicited-node-multicast/routerA-2address.png"></p> 

再看接口信息，这时就会会发现它还加入了另一个 solicited-node 地址。

<p align="center"><img src="/assets/img/ipv6-solicited-node-multicast/join-text-2.png"></p> 

**为什么?**  

为什么设备需要为每一个 IPv6 单播地址计算 solicited-node 组播地址，并加入对应的组播组呢？简单地说，就是为了给邻居发现协议使用([RFC 4861](http://datatracker.ietf.org/doc/rfc4861/)).

在之后的博文中，我会解释 solicited-node 组播地址运用场景。
