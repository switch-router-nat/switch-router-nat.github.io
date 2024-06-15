---
layout    : post
title     : "[译]理解IPv6：Link-Local地址的魔法"
date      : 2018-10-13
lastupdate: 2018-10-13
categories: Translate
---

原文链接：[Understanding IPv6: Link-Local 'Magic'](https://www.networkcomputing.com/networking/understanding-ipv6-link-local-magic)

------------------

鉴于你是 IPv6 的初学者，下面我将为你展示一个小魔术：我将在两台路由器之间建立 IPv6 IGP 邻居关系(使用 OSPFv3 路由协议)。这乍听起来毫无新意，但我如果告诉你，**我不会在它们上配置任何 IPv6 地址，它们之间的邻居关系也能成功建立成功**，你还这么认为吗？

<p align="center"><img src="/assets/img/ipv6-link-local/router-1.jpg"></p>

下面是我为这个"魔术"做的准备工作：

- 两台路由器都使能 IPv6 的单播路由
- 两台路由器都通过命令 "ipv6 router ospf 6" 使能 IPv6 的 OSPFv3 路由协议
- 两台路由器都配置了一个管理用的 IPv4 的地址，它们都在 14.14.14.0/24 网段
- 没有配置任何 IPv6 地址
- 两台路由器相连的接口均为 Gig1/0/1 ，它们都只配置了两条 IPv6 相关配置, 处于 shut down 状态
- 路由器 A 会抓取 Gig1/0/1 接口上的流量，并将流量发送给测试仪。


<p align="center"><img src="/assets/img/ipv6-link-local/router-2.jpg"></p>

好了，现在我通过 "no shut" 命令将路由器 A 和路由器 B 的接口连接起来，然后念念咒语....

然后，看！ OSPFv3 的邻居建立起来了！

<p align="center"><img src="/assets/img/ipv6-link-local/images-4.jpg"></p>

**猜猜看是怎么发生的？**

我是如何做到的？ 回忆一下我在上一篇 [blog](http://www.networkcomputing.com/networking/understanding-ipv6-the-journey-begins/a/d-id/1279120) 中提到的问题：

*本地链路上地址分配: 为什么我们要将珍贵的 IP 地址浪费在路由器在本链路中互相通信时发送本地报文？这些报文仅仅存在于本链路上啊*

这是一个好问题！考虑到 IPv4 地址已经被用尽，在 IPv6 中，人们提前了考虑地址保留技术用于这种场景。那么，具体是怎么做到的呢？

**Sniffer 抓到的报文**

我们从邻居建立过程中的报文交互情况开始入手：

<p align="center"><img src="/assets/img/ipv6-link-local/Sniffer-trace-1.jpg"></p>

瞧! 如此多的 FE80:: 和 FF02::5 地址！

这些地址可能略显杂乱，难以阅读。但你仔细看会发现实际上地址栏里都只是下面这三个地址来回变化：

- FF02::5
- FE80::5A0A:20FF:FEEB:91E4
- FE80::2237:06FF:FECF:67E4

它们是什么意思？

**FF02::5**

首先是 FF02::5

<p align="center"><img src="/assets/img/ipv6-link-local/sniffer-2.jpg"></p>

它的后缀 "::5" 有没有让你想到什么？还有，注意到了吗？它始终只出现在目的地址那一栏，而不会在源地址那一栏！如果我提到 IPv4 地址 224.0.0.5，你会把它们联系咋一起吗？它们的确是有关系的！

看看[this IANA page](http://www.iana.org/assignments/multicast-addresses/multicast-addresses.xhtml#multicast-addresses-1), 和[other IANA page](http://www.iana.org/assignments/ipv6-multicast-addresses/ipv6-multicast-addresses.xhtml#link-local)表格。

<p align="center"><img src="/assets/img/ipv6-link-local/IPv6-vs-IPv4.jpg"></p>

FF02::5 的谜底揭开了！它正是 IPv6 中等价于 IPv4 中的 OSPF 组播目的地址 224.0.0.5 

**FE80::**

这些 FE80:: 的 IPv6 地址从哪来的？显然，它们就是路由器 A 和路由器 B 的。

<p align="center"><img src="/assets/img/ipv6-link-local/sniffer-4.jpg"></p>

问题是，我从来没有配置过这些地址呀？！好了，下面允许我隆重介绍 IPv6 link-local 单播地址类型。

RFC 4291 的 2.4 节介绍了多种地址类型。刚才的 FF02::5 正是属于其中 FF00::/8 这一类。而现在我们关注的 FE80:: 则是属于 link-local 单播这一类。

<p align="center"><img src="/assets/img/ipv6-link-local/address-type-verification.jpg"></p>

支持 IPv6 的主机的每个接口都必须有一个 link-local 单播地址。这个 link-local 地址都以 FE80:: 开头。但 FE80:: 后面的数字从何而来呢？

<p align="center"><img src="/assets/img/ipv6-link-local/router-3.jpg"></p>

看看上面的 MAC 地址，再看看两个 link-local 地址
- FE80::5A0A:20FF:FEEB:91E4
- FE80::2237:06FF:FECF:67E4

发现端倪了吗？

<p align="center"><img src="/assets/img/ipv6-link-local/router-4.jpg"></p>

我所使用的路由器使用 RFC4291 附录 A 中描述的方法用 48-bit MAC 地址生成 link-local 地址。具体细节可以参考 RFC 标准。

**静态配置 IPv6 link-local 地址**

我们也可以手动覆盖自动生成的 link-local 地址。举个栗子，我可以为路由器 A 的接口配置FE80::1 ，而为路由器 B 的接口配置 FE80::2。

<p align="center"><img src="/assets/img/ipv6-link-local/router-5.jpg"></p>

同样的操作，此时的抓包结果的可读性就好多了。

<p align="center"><img src="/assets/img/ipv6-link-local/sniffer-5.jpg"></p>

**结语**

从以上的内容中可以看出，上面 IPv6 IGP 邻居的建立并不需要我们配置 IPv6 的全球单播地址。也许你会有疑问：“如果我为接口配置了全球单播地址，那么 IGP (OSPFv3) 是会使用全球单播地址作为源地址和目的地址吗？”

很好的问题！我会在以后的博客中回答这个问题:)

