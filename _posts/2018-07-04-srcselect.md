---
layout    : post
title     : "Linux 报文源地址选择那点事儿"
date      : 2018-07-04
lastupdate: 2018-07-04
categories: Network(Kernel)
---

源地址和目的地址是 IP 首部中最重要的两个字段，而我们总是习惯于关注报文的目的地址，而忽略源地址。这完全是情有可原的！因为作为开发者来说，关心的是数据往哪发，作为运维者来说，配置路由规则也是按目的地址进行配置。至于源地址？就让内核自己去搞定好了，而内核似乎也真的能搞定。 

源地址选择不重要吗？当然不是，源地址在绝大多数情况下就是对端的目的地址，它的选择是否正确决定了对端能不能将该响应正确地送回来。对于只有一个 IP 地址(排除 loopback )的主机来说，那没得说，只能选它；但如果有多个 IP 地址呢？ 要知道一台主机可以有多个网卡，每个网卡都可以配置多个 IP 地址，这时源地址该如何选择呢？

直接给出答案吧，内核按照以下顺序尝试选择：

1. socket 层次，用户使用 bind() 绑定的 IP 地址
2. 查询路由时，如果匹配的路由带有 src 关键字，则使用其指定的地址
3. 查询路由时，得到的出接口上配置的同网段的第一个 primary 地址
4. 查询路由时，其他接口上配置的 primary 地址

看上去源地址是完全在网络层完成的事，但实际上，不同的传输层( TCP 和 UDP )也会对源地址选择产生影响。   

#### TCP 源地址选择

TCP 是面向连接的，这里的面向连接隐含一层意思：连接的四元组一旦建立，就不会变了。

对于发起端，它会在发送第一个 SYN 报文时(也就是用户使用 connect() 时)就进行源地址选择。而对于被动端则很简单，直接使用 SYN 报文中的目的地址。它需要重新源地址选择吗？不需要，或者说不能！原因是四元组唯一确定一条 TCP 连接，既然发起端已经在 SYN 中定下了<源IP，目的IP>，那么被动端只能默默接受，否则这怎么能算同一条连接呢？

内核相关实现：
主动端
```c
int tcp_v4_connect(struct sock *sk, struct sockaddr *uaddr, int addr_len)
{
    ......
    if (!inet->inet_saddr)
		inet->inet_saddr = fl4->saddr; 
}
```

被动端：
```c
// 收到 SYN 请求时
static void tcp_v4_init_req(struct request_sock *req,
			    const struct sock *sk_listener,
			    struct sk_buff *skb)
{
    ......
    sk_daddr_set(req_to_sk(req), ip_hdr(skb)->saddr);
}

// 收到 ACK 时
struct sock *tcp_v4_syn_recv_sock(const struct sock *sk, struct sk_buff *skb,
				  struct request_sock *req...
{
    ....
    newinet->inet_saddr	      = ireq->ir_loc_addr;
}                  
```


#### UDP 源地址选择

UDP 在很多方面都没有 TCP 复杂，但源地址选择是个例外。UDP 没有数据流或者连接的概念，因此他不必听命于对端的报文的目的地址作为源地址，而是可以另起炉灶，重新选择。比如在下面的拓扑中

<p align="center"><img src="/assets/img/srcselect/udp.PNG"></p>

HOST 2 作为 UDP Server, 当 HOST 1 发送一个源地址 为 10.0.0.1 且目的地址为 192.168.2.1 的 UDP 报文后，HOST 2 查询路由发现回复的报文应该走 eth1，因此回复的报文源地址为 192.168.3.1 

有什么办法可以避免 UDP 每个报文都去进行路由查找然后源地址选择呢？答案是 bind() 或者 connect() 

connect() 的相关代码如下
```c
int __ip4_datagram_connect(struct sock *sk, struct sockaddr *uaddr, int addr_len)
{
    ......
    if (!inet->inet_saddr)
		inet->inet_saddr = fl4->saddr;	/* Update source address */
}
```
