---
layout    : post
title     : "IPsec与NAT Traversal(NAT-T)"
date      : 2019-09-14
lastupdate: 2019-09-14
categories: Network(Kernel)
---

<p align="center"><img src="/assets/img/IPsec-nat-t/travalsal.jpg"></p>
## 背景

IPsec在两个通信实体之间建立安全的数据传输通道, 但它却与网络中广泛存在的NAT设备(以及PAT)有天生的不兼容性(incompatible)。

我们以一个TCP报文为例来看看在不同IPsec的不同模式(Transport和Tunnel)和协议(AH和ESP)下，这种不兼容是如何发生的。

先来看Transport模式

<p align="center"><img src="/assets/img/IPsec-nat-t/ipsec-transport.PNG"></p>

对AH协议，由于其Authenticate范围是整个IP报文，所以如果两个IPsec之间存在NAT设备，修改了报文IP Header中的地址，就会导致接收方的Authenticate失败。
对ESP协议，其Authenticate返回不包括IP Header,所以接收方的Authenticate会通过，但如果中间的NAT设备修改了IP Header中的地址，理论上后面的TCP checksum也会随之修改，但这部分在ESP协议中是
加密的，NAT设备没有办法修改，所以接收端在TCP接收时会出现checksum校验失败。

再来看Tunnel模式
<p align="center"><img src="/assets/img/IPsec-nat-t/ipsec-tunnel.PNG"></p>
对AH协议, Tunnel模式和Transport模式没什么不同，Authenticate范围包含了外层IP Header，因此同样会造成接收方Authenticate失败。
对ESP协议，与Transport模式不同的是，经过NAT设备。内层IP Header并不会改变，所以TCP checksum也不会变化，接收方不会出现checksum校验失败。

这样看起来，ESP-Tunnel似乎成为了在有NAT设备环境下，唯一可行的协议-模式组合。但即使是这种组合也是有缺点的：它只能支持一对一的NAT(NAT设备后面只有一台内网主机)。在很多组网中，NAT设备通常作为网关使用，其背后可能有很多台主机。这时地址转换就不够了，它还需要端口转换，显然，NAT设备对ESP-Tunnel的报文是无能为力的，因为TCP部分已经被加密了，已经没有端口字段了。
所以，IPsec需要想办法能绕开NAT设备的影响，也就是进行NAT穿越(NAT-Tranversal)。

## UDP-Encapsulate

IPsec采用的办法是在ESP Header前加上一个UDP Header, 这个方法同时适用于ESP-Transport和ESP-Tunnel模式。
下图展示了ESP-Transport下的UDP-Encapsulate过程。

<p align="center"><img src="/assets/img/IPsec-nat-t/udp_encap.PNG"></p>
UDP Header是有端口字段的，有了端口，NAT设备便可以进行端口转换。RFC3948中规定UDP Header中的端口要使用和IKE协商时相同的端口号，这个端口号在RFC3947中规定为4500.

在下面这样的拓扑中，NAT设备背后有两台内网主机，它们都与Server建立IPsec连接。

<p align="center"><img src="/assets/img/IPsec-nat-t/pat.PNG"></p>
Host 1与Host 2发出的IPsec报文都附加了一个UDP Header。NAT网关替换该报文的Source IP和Source Port。

还有一个问题, 对于ESP-Transport模式, 内层TCP报文的checksum校验的问题如何解决呢？要知道，经过NAT设备之后，报文的IP地址发生了变化，这势必导致接收端校验失败。IPsec采用的方法是在IKE协商时，就将自己原始IP地址信息发给对端，这样Server在解密出TCP报文后，可以根据这个信息修正checksum

## NAT-T 协商过程

IPsec的通信实体之间需要在IKE时完成协商才能使用上面UDP-Encapsulate，完成NAT-T。

在IKE的PHASE1

- 双方探测出双方都支持NAT-T
- 双方探测出了报文传输路径上存在NAT设备

在IKE的PHASE2

- 双方协商NAT-T的封装模式，UDP-Encapsulated-Tunnel还是UDP-Encapsulated-Transport
- (UDP-Encapsulated-Transport)模式向对方发送自己的原始IP地址, 让对方可以据此修正后续TCP报文的checksum

为了更好的说明我用虚拟机搭建了下面这个拓扑，用来展示IPsec的NAT-T协商过程

<p align="center"><img src="/assets/img/IPsec-nat-t/topo.PNG"></p>
其中Alice和Carol上运行Strongswan, 而Moon作为NAT设备。配置IPsec为Transport模式，使用IKEv1进行协商

### 探测支持 NAT-T

IPsec的两端在PHASE1的消息1和消息2中会通过交换vendor ID payload来向对方通告自己支持NAT, 其内容正是字符串"rfc3947"

<p align="center"><img src="/assets/img/IPsec-nat-t/ike-msg1.png"></p>
### 探测是否存在 NAT

在IKE PHASE1的消息3和消息4，通信双方会交换自己的和自己眼中对方的IP和Port的哈希值，如果中间存在NAT设备，则该值一定与该报文本身的IP和Port计算出的值不一致。

<p align="center"><img src="/assets/img/IPsec-nat-t/ike-msg3.png"></p>
### 改变端口从500到4500

IKE PHASE1的前4个消息都是使用Sport=Dport=500进行通信。但当探测到NAT设备存在时，作为Initiator的Alice就再消息5需要将端口切换到Sport=Dport=4500, 作为Responder的Carol在收到该消息后，如果解密成功，也会使用新的4500端口

<p align="center"><img src="/assets/img/IPsec-nat-t/ike-4500.png"></p>
在此之后，后续的IKE PHASE2和业务流量都会使用4500端口进行UDP-Encapsulate。为了与业务流量进行区分，IKE阶段的流量紧随UDP Header后的是一个32bit全为0的Non-ESP Marker (业务流量的这个地方是填写的是非零的SPI)

<p align="center"><img src="/assets/img/IPsec-nat-t/ike-non-ESP.png"></p>

## 内核相关实现

内核使用xfrm框架完成IPsec报文收发功能。普通情况下, IP根据协议字段分流IPsec报文和TCP UDP报文。

<p align="center"><img src="/assets/img/IPsec-nat-t/ip_rcv.PNG"></p>

在NAT-T场景中，IPsec为报文进行了UDP-Encapsulate，那么，接收端看到的就是一个UDP报文了，会调用udp_rcv()进行报文接收。那么此时又如何进入xfrm框架呢？

答案是：Strongswan通过设置UDP套接字UDP_ENCAP选项，内核为套接字绑定一个回调函数xfrm4_udp_encap_rcv()

```c
int udp_lib_setsockopt(struct sock *sk, int level, int optname,
		       char __user *optval, unsigned int optlen,
		       int (*push_pending_frames)(struct sock *))
{
    // code omitted
    switch (optname) {
        case UDP_ENCAP:
        switch (val) {
            case 0:
            case UDP_ENCAP_ESPINUDP:
            case UDP_ENCAP_ESPINUDP_NON_IKE:
                up->encap_rcv = xfrm4_udp_encap_rcv;
        // code omitted
    }
}
```

而在udp_rcv()接收过程中，最终会调用到该回调函数

<p align="center"><img src="/assets/img/IPsec-nat-t/udp_decap.PNG"></p>

## REF

[RFC 3947 Negotiation of NAT-Traversal in the IKE](https://tools.ietf.org/html/rfc3947) 
[RFC 3948 UDP Encapsulation of IPsec ESP Packets](https://tools.ietf.org/html/rfc3948)
 































