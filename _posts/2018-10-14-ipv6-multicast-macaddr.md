---
layout    : post
title     : "[译]理解IPv6：组播MAC地址"
date      : 2018-10-14
lastupdate: 2018-10-14
categories: Translate
---

原文地址:[understanding-ipv6-sniffer-full-3s](https://www.networkcomputing.com/cloud-infrastructure/understanding-ipv6-sniffer-full-3s)

-----------------------------

"这么多数字 3 是什么鬼？" 当我第一次看到下面这张图时，我发出这种疑问....

<p align="center"><img src="/assets/img/ipv6-multi-macaddr/3sniffer-trace.png"></p>

这不禁让我想起了令牌环网络中的本地管理地址(LAAs)。我这么想的原因有二：

其一，这些奇怪的 MAC 地址都是充当的是目的地址，而不是源地址，这很像令牌环中的 LAAs.
其二，这些 MAC 地址看上去太整洁了，就像令牌环 LAA 中为 3745 IBM 前端进程使用的 4000.3745.0001。看看这些 MAC 地址，它们都是是由 4 个 3 开头，跟着一大堆 0，最后是一个很小的数字。 

**使用 Wireshark**

Wireshark 能将线缆上的报文一五一十地展现出来，它在解决网络问题中十分有用。如果你很熟悉它，你应该知道你可以在软件的首选项(preferences)界面设置显示报文的 MAC 地址，就向下图中展示的这样，我设置Wireshark 显示每个报文的 MAC 地址，注意，我保持它们为 unresolved 的状态。

<p align="center"><img src="/assets/img/ipv6-multi-macaddr/wireshark-preferences.png"></p>

如果按照以上的设置，那么抓包结果就将像下面这样被显示出来了：

<p align="center"><img src="/assets/img/ipv6-multi-macaddr/mac-address1.png"></p>

在我们更进一步前，我想问你一个问题。你能看出目的 MAC 33:33 所在的报文有什么特点吗 ? 仔细点！

<p align="center"><img src="/assets/img/ipv6-multi-macaddr/mac-address2.png"></p>

这些报文都对应 FF02:: 这样的 IPv6 地址，这些地址又是什么鬼 ?

[RFC 4291](https://datatracker.ietf.org/doc/rfc4291/)的 2.4 小节列举出了各种各样的 IPv6 地址类型。就像我在上一篇[博客]()中所写的，FF00::/8 组播地址。事实上，如果你还记得的话，FF02::5 和 FF02::6 是 OSPF 专用的组播地址。

<p align="center"><img src="/assets/img/ipv6-multi-macaddr/wireshark-preferences.png"></p>

现在看出来 33:33 是什么了吗？

如果还没有，那就将刚才 Wireshark 首选项中的显示 MAC 地址设置为 resolve 吧。

<p align="center"><img src="/assets/img/ipv6-multi-macaddr/wireshark-modified-preferences.png"></p>

再来看抓包结果：

<p align="center"><img src="/assets/img/ipv6-multi-macaddr/mac-address-results3.png"></p>

好了，现在明白的吧？ Wireshark 将刚才的 "33:33" 变成了 "IPv6mcast" ！

IPv6 目的地址 FF02::5 对应 IPv6mcast_00:00:00:00:05
IPv6 目的地址 FF02::6 对应 IPv6mcast_00:00:00:00:06

**从 RFC 中找寻答案**

[RFC 7042](https://datatracker.ietf.org/doc/rfc7042/) 的 2.3.1 小节是这么写的：

> All MAC-48 multicast identifiers prefixed "33-33" (that is, the 2**32 multicast MAC identifiers in the range from 33-33-00-00-00-00 to 33-33-FF-FF-FF-FF) are used as specified in [RFC2464] for IPv6 multicast. In all of these identifiers, the Group bit (the bottom bit of the first octet) is on, as is required to work properly with existing hardware as a multicast identifier. They also have the Local bit on and are used for this purpose in IPv6 networks.
>

RFC 将 33-33-00-00-00-00 到 33-33-FF-FF-FF-FF 这段 MAC 地址保留给 IPv6 的组播报文使用。现在的问题变成了，为什么 FF02::5 对应 IPv6mcast_00:00:00:00:05 ?

[RFC 2464](https://datatracker.ietf.org/doc/rfc2464/)的第 7 节解释了组播地址的映射规则：

一个目的地址为组播地址的 IPv6 报文，在线路上传输时，目的 MAC 地址的前 2 个字节为十六进制的 0x3333 ，后 4 个字节为 IPv6 目的地址的最后 4 个字节。

<p align="center"><img src="/assets/img/ipv6-multi-macaddr/DST.png"></p>

**与 IPv4 差异很大?**

IPv6 的这种行为和 IPv4 相差很大 ? 不！你之所以觉得大只是因为你更熟悉 IPv4 的 MAC 地址。实际上, IPv4 使用的组播 MAC 地址只是显得不那么特殊而已 (IPv4 使用 01:00:5e, IPv6 使用 33:33)

下面是 IPv4 的组播报文的 Wireshark 抓包结果, 这里的 MAC 地址是 unresolved 的

<p align="center"><img src="/assets/img/ipv6-multi-macaddr/unresolved-view.png"></p>

是不是和 IPv6 的很像 ? 以 "01:00:5e" 开头的 MAC 地址只会出现在目的 MAC 地址那一栏，并且只有当目的 IPv4 地址是组播地址时 (224/8)

下面是 resolved 的形式显示结果：

<p align="center"><img src="/assets/img/ipv6-multi-macaddr/resolved-view.png"></p>
