---
layout    : post
title     : "什么是透明代理"
date      : 2019-11-14
lastupdate: 2019-11-14
categories: Network(others)
---

#### 引

多年前还在学校读本科时，为了节省网费，常去内网论坛用虚拟货币"买代理"，然后就可以用它外网了！对于一个穷学生，真是莫大的福音！

多年之后，在工作岗位上，我竟然再次与代理产生了联系。那就索性总结一下吧。不过本文依旧是科普性质，与工作内容无关:)

#### 什么是代理

本文的题目是透明代理( Transparent Proxy ), 所以我们必须首先搞清楚什么是代理。假设 A 与 B 要通信，如果它们之间有一个中间人 C 当传话筒，那么 C 就是代理。

我们常常还会听到两个词叫前向代理( Forward Proxy) 和反向代理 ( Reverse Proxy )。区分它们主要看**代理在请求中扮演的角色是客户端还是服务器**。

##### 前向代理 VS 反向代理

<p align="center"><img src="/assets/img/transparent-proxy/forward-reverse.png"></p>

像"买代理上外网"，或者"使用 shadowsock 科学上网"的方式都属于前向代理。在用户使用的过程中，代理服务器扮演的是客户端的角色，即代理代替客户端向服务器发送请求，然后将服务的回应返回给客户端.

而在反向代理中，代理扮演的是服务器的角色，它背后是真正的后端服务器(甚至是集群)，用户只知道它访问的是代理，却不知道它的请求最终会到达后端服务器。比如著名的 Nginx 常常会用来作反向代理。

#### 什么是透明代理

那透明代理又是什么呢？从"透明"这个词也可以猜出一二了，透明意味着代理本身对用户是不可见( invisible )的，也就是说用户侧不需要进行代理设置(区别于前向代理)，请求的目标也是真实的服务器(区别于反向代理)。作为中间人，透明代理会拦截用户的请求，其常见的应用场景有：

- 缓存请求的结果。透明代理缓存服务器的回应，这样之后重复请求到达时，直接回复给用户就好了，不用再将请求扔给服务器，这一点有没有让你想到 CDN ?

- 过滤请求。也就是一些公司常常使用的行为管理功能，禁止掉员工对 QQ、weixin、炒股等网站的请求，

#### Linux 实现透明代理

Linux 通过 IP_TRANSPARENT 和 TPROXY 可以轻松地实现透明代理。它们实现了以下特性：

- 重定向报文
- 监听不属于本机的地址，以不属于本机的地址向外发起连接

##### 重定向报文

我们知道，当 Linux 从网卡收到报文时会查询路由( fib )，路由决定了报文是应该**转发** ( forward )还是**上送本机** ( local deliever )。既然是代理，我们自然希望感兴趣的报文能上送本机，而不是直接转发走。如何做呢？我们知道内核决定是否上送本机取决于报文的目的地址是否是本地( local )地址，比如 127.0.0.0/8 这个网段的就是本地地址。

这样还不够，作为代理，它需要把感兴趣的用户的报文截获,即**明知道这报文不是给自己的，也要收上来**，所以它需要将更多的地址视为本地地址，比如按照如下配置，可以将所有地址(0.0.0.0/0)这个视为本地地址
```
ip route add local 0.0.0.0/0 dev lo 
```
但这样配置不好的一点在于，整个主机变成了"黑洞"，所有地址都是本地地址，报文就发不出去了。因此，我们需要结合策略路由，让代理单独使用一张路由表。
```
iptables -t mangle -I PREROUTING -p tcp --dport 5301 -j MARK --set-mark 1
ip rule add fwmark 1 lookup 100
ip route add local 0.0.0.0/0 dev lo table 100
```
上面的意思是说，将所有收到的目的端口为 5301 的数据包打上标记 ‘1’。然后添加一条策略路由，对有标记 ‘1’ 的数据包查询路由时使用路由表 table 100，而在表 table 100 中添加的唯一一条路由就是将所有地址都视为本地地址。

##### 监听外部地址

我们知道，通常用户服务器程序使用 bind() 绑定本机地址时，需要这个地址属于本机或者干脆为 INADDR_ANY。而我们作为透明代理，需要 bind() 绑定真正服务器的 IP 地址，这时  IP_TRANSPARENT 这个 socket 选项可以派上用场了。

```c
int val = 1;
setsockopt(fd, SOL_IP, IP_TRANSPARENT, &val, sizeof(val));
```
设置了 IP_TRANSPARENT 选项后，bind() 将不会检查填入的地址是否是本地地址。假设我的服务器地址是 1.2.3.4 , 那这里我就可以监听 1.2.3.4。

结合上一节的策略路由的设置，我们的代理 socket 就可以收到用户发送的请求了，比如我们可以用 telnet 进行测试。
```
telnet 1.2.3.4 80
```
##### TPROXY

我们还可以使用 iptables 另一个扩展 `TPROXY` [扩展](http://ipset.netfilter.org/iptables-extensions.man.html)来继续优化代理服务器。

前面设置策略路由的方法有一个缺陷，那就是当代理程序关闭时，目标端口为 5301 的报文还是会源源不断地上送，而它们最后会应为找不到 socket 而被丢弃。使用 TPROXY 可以让这个过程提前，避免 FIB 查询过程。
```
iptables -t mangle -A PREROUTING -p tcp --dport 5301 -j TPROXY --tproxy-mark 0x1/0x1 --on-port 8888 --on-ip 1.2.3.4
```

这条规则的意思是：在收到报文的 PREROUTING 阶段(也就是查询 FIB 前)，如果它的目的端口是 5301 ，则 iptables 会立即查找本机是否有建立在 1.2.3.4:80 的 socket。若有(代理程序运行)，为其打上一个标记( 0x1 )，并且报文在通过 FIB 查找然后上送，会由拥有该 socket 的应用程序处理。若没有(代理程序关闭)，在这个阶段就将报文丢弃，避免路由查找浪费。

### REF

- [What's the difference between a proxy server and a reverse proxy server?](https://stackoverflow.com/questions/224664/whats-the-difference-between-a-proxy-server-and-a-reverse-proxy-server)
- [https://powerdns.org/tproxydoc/tproxy.md.html](https://powerdns.org/tproxydoc/tproxy.md.html)










