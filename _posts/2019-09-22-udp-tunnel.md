---
layout    : post
title     : "Linux内核中的udp隧道框架"
date      : 2019-09-22
lastupdate: 2019-09-22
categories: Network(Kernel)
---

<p align="center"><img src="/assets/img/public/tunnel.png"></p>

## 起源

TCP虽然能保证传输的可靠性，但其繁琐的状态机以及复杂的拥塞控制机制让它难以作为隧道报文的外层封装.

相对而言，UDP就没这个困扰了，丢包的事情交给应用层处理就行。因而，不少隧道协议都是将UDP作为外层报文的方案。自然而然，与网络发展联系紧密的Linux内核也开始支持这些隧道协议，较新的内核已经支持fou、l2tp、vxlan、tipc、geneve等UDP隧道协议。

最开始，各个隧道协议都是独立实现的，但随着数量的增多，在[patch](https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/commit/net/ipv4/udp_tunnel.c?id=8024e02879ddd5042be02c70557f74cdc70b44b4)之后，内核将这些UDP隧道公共的部分抽离出来，也就形成了UDP隧道框架，其涉及的API在`include/net/udp_tunnel.h`中定义

## 内核UDP隧道的原理

下图以vxlan为例，展示了内核UDP隧道的工作过程：

<p align="center"><img src="https://s2.ax1x.com/2019/09/22/upkkuT.png"></p>
其中，左边是发送端，右边是接收端，绿色阴影的部分是内核协议栈。可以看出，无论是发送端还是接收端，都涉及**函数重入**：发送端两次进入`ip_local_out()`, 接收端两次进入`ip_local_deliver()`

### 隧道socket

对发送端来说，第一次进入`ip_local_out()`传入的`sk`是与原始报文关联的套接字，也就是原始协议的套接字，它可能是个TCP套接字，也可能是UDP套接字或者RAWIP套接字，隧道并不care这件事。但是第二次进入`ip_local_out()`时，它需要一个隧道的UDP套接字。UDP隧道框架提供了一个创建隧道套接字的API。

```c
static inline int udp_sock_create(struct net *net,
				  struct udp_port_cfg *cfg,
				  struct socket **sockp)
{
    if (cfg->family == AF_INET)
        return udp_sock_create4(net, cfg, sockp);

    ......
    return -EPFNOSUPPORT;
}
```
`cfg`参数指定了UDP隧道本端和对端和IP地址和使用的端口号。这里创建的套接字都是**内核套接字**(区别于用户态使用socket()创建的)

接收端也是同样的道理，从真实网卡收到的一定是一个UDP报文，因此接收端也需要一个UDP套接字。这个套接字中记录的地址和端口信息与接收端正好相反。

<p align="center"><img src="https://s2.ax1x.com/2019/09/22/upkivV.png"></p>
### Encap rcv回调函数

对接收端来说，收到UDP报文后，它还需要将报文找到分流给正确的隧道协议，比如这个UDP隧道报文是交给vxlan，还是交给geneve？又或者这根本就只是一个普通的UDP报文，不是一个UDP隧道报文？

因此，内核需要将如何分流记录在UDP套接字上。

```c
struct udp_sock {
    ......
    /*
     * For encapsulation sockets.
     */
    int (*encap_rcv)(struct sock *sk, struct sk_buff *skb);
    ......
};
```
这里的`encap_rcv`回调函数便是起到隧道报文分流的作用

在UDP接收时，内核会首先查看套接字上是否设置了该回调函数，如果设置了，表示这是一个隧道套接字。调用对应的处理函数，比如vxlan隧道会将其设置为`vxlan_rcv`,genven隧道会将其设置为`geneve_udp_encap_recv`。

设置的过程是通过下面这个API完成的
```c
void setup_udp_tunnel_sock(struct net *net, struct socket *sock,
			   struct udp_tunnel_sock_cfg *cfg)
```

