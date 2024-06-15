---
layout    : post
title     : "backlog参数对TCP连接建立的影响"
date      : 2020-05-21
lastupdate: 2020-05-21
categories: Network(Kernel)
---
> 曾经有人问我套接字编程中`listen`的第二个参数`backlog`是什么意思？多大的值合适？我不假思索地回答它表示服务器可以接受的并发请求的最大值。然而事实真的是这样的吗？

<p align="center"><img src="/assets/img/tcp-backlog/status-convert.png"></p>

`TCP`通过三次握手建立连接的过程应该都不陌生了。从服务器的角度看，它分为以下几步

 1. 将`TCP`状态设置为`LISTEN`状态，开启监听客户端的连接请求
 2. 收到客户端发送的`SYN`报文后，`TCP`状态切换为`SYN RECEIVED`，并发送`SYN ACK`报文
 3. 收到客户端发送的`ACK`报文后，`TCP`三次握手完成，状态切换为`ESTABLISHED`

在`Unix`系统中，开启监听是通过`listen`完成。

```
int listen(int sockfd, int backlog)
```
`listen`有两个参数，第一个参数`sockfd`表示要设置的套接字，本文主要关注的是其第二个参数`backlog`；

<<Unix 网络编程>> 将其描述为**已完成的连接队列**(`ESTABLISHED`)与**未完成连接队列**(`SYN_RCVD`)之和的上限。

一般我们将`ESTABLISHED`状态的连接称为**全连接**，而将`SYN_RCVD`状态的连接称为**半连接**

<p align="center"><img src="/assets/img/tcp-backlog/listen.png"></p>
当服务器收到一个`SYN`后，它创建一个**子连接**加入到`SYN_RCVD`队列。在收到`ACK`后，它将这个**子连接**移动到`ESTABLISHED`队列。最后当用户调用`accept()`时，会将连接从`ESTABLISHED`队列取出。


----------

### 是 Posix 不是 TCP

`listen`只是`posix`标准，不是`TCP`的标准！不是`TCP`标准就意味着不同的内核可以有自己独立的实现

[POSIX是这么说的][3]:

> The `backlog` argument provides a hint to the implementation which the implementation shall use to limit the number of outstanding connections in the socket's listen queue.


`Linux`是什么行为呢 ? 查看`listen`的`man page`

> The  behavior  of  the  backlog  argument on TCP sockets changed with Linux 2.2.  Now it specifies the queue length for completely established sockets waiting to be accepted, instead of the number of incomplete connection requests. 

什么意思呢？就是说的在`Linux 2.2`以后, `backlog`只限制完成了三次握手，处于`ESTABLISHED`状态等待`accept`的子连接的数目了。

真的是这样吗？于是我决定抄一个小程序验证一下：

服务器监听`50001`端口，并且设置`backlog = 4`。注意，我为了将队列塞满，没有调用`accept`。

```c
#include <stdio.h>
#include <unistd.h>
#include <string.h>
#include <sys/socket.h>
#include <netinet/in.h>

#define BACKLOG 4

int main(int argc, char **argv)
{
    int listenfd;
    int connfd;
    struct sockaddr_in servaddr;

    listenfd = socket(PF_INET, SOCK_STREAM, 0);

    bzero(&servaddr, sizeof(servaddr));
    servaddr.sin_family = AF_INET;
    servaddr.sin_addr.s_addr = htonl(INADDR_ANY);
    servaddr.sin_port = htons(50001);

    bind(listenfd, (struct sockaddr *)&servaddr, sizeof(servaddr));

    listen(listenfd, BACKLOG);
    while(1)
    {
        sleep(1);
    }
    
    return 0;
}

```

客户端的代码
```c
#include <stdio.h>
#include <string.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <unistd.h>

int main(int argc, char **argv)
{
    int sockfd;
    struct sockaddr_in servaddr;

    sockfd = socket(PF_INET, SOCK_STREAM, 0);

    bzero(&servaddr, sizeof(servaddr));
    servaddr.sin_family = AF_INET;
    servaddr.sin_port = htons(50001);
    servaddr.sin_addr.s_addr = inet_addr("127.0.0.1");

    if (0 != connect(sockfd, (struct sockaddr *)&servaddr, sizeof(servaddr)))
    {
         printf("connect failed!\n");
    }
    else
    {
         printf("connect succeed!\n");
    }

    sleep(30);

    return 1;
}
```

为了排除`syncookie`的干扰，我首先关闭了`syncookie`功能
```
echo 0 > /proc/sys/net/ipv4/tcp_syncookies
```

由于我设置的`backlog = 4`并且服务器始终不会`accept`。因此预期会建立 **4** 个全连接, 但实际却是
```
root@ubuntu-1:/home/user1/workspace/client# ./client &
[1] 12798
root@ubuntu-1:/home/user1/workspace/client# connect succeed!
./client &
[2] 12799
root@ubuntu-1:/home/user1/workspace/client# connect succeed!
./client &
[3] 12800
root@ubuntu-1:/home/user1/workspace/client# connect succeed!
./client &
[4] 12801
root@ubuntu-1:/home/user1/workspace/client# connect succeed!
./client &
[5] 12802
root@ubuntu-1:/home/user1/workspace/client# connect succeed!
./client &
[6] 12803
root@ubuntu-1:/home/user1/workspace/client# connect succeed!
./client &
[7] 12804
root@ubuntu-1:/home/user1/workspace/client# connect succeed!
./client &
[8] 12805
root@ubuntu-1:/home/user1/workspace/client# connect succeed!
./client &
[9] 12806
root@ubuntu-1:/home/user1/workspace/client# connect succeed!
./client &
[10] 12807
root@ubuntu-1:/home/user1/workspace/client# connect failed!
```

看！客户器竟然显示成功建立了 **9** 次连接！

用`netstat`看看`TCP`连接状态

```
> netstat -t
Active Internet connections (w/o servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State
tcp        0      0 localhost:50001         localhost:55792         ESTABLISHED
tcp        0      0 localhost:55792         localhost:50001         ESTABLISHED
tcp        0      0 localhost:55798         localhost:50001         ESTABLISHED   
tcp        0      1 localhost:55806         localhost:50001         SYN_SENT
tcp        0      0 localhost:50001         localhost:55784         ESTABLISHED
tcp        0      0 localhost:50001         localhost:55794         SYN_RECV
tcp        0      0 localhost:55786         localhost:50001         ESTABLISHED
tcp        0      0 localhost:55800         localhost:50001         ESTABLISHED
tcp        0      0 localhost:50001         localhost:55786         ESTABLISHED
tcp        0      0 localhost:50001         localhost:55800         SYN_RECV
tcp        0      0 localhost:55784         localhost:50001         ESTABLISHED
tcp        0      0 localhost:50001         localhost:55796         SYN_RECV
tcp        0      0 localhost:50001         localhost:55788         ESTABLISHED
tcp        0      0 localhost:55794         localhost:50001         ESTABLISHED
tcp        0      0 localhost:55788         localhost:50001         ESTABLISHED
tcp        0      0 localhost:50001         localhost:55790         ESTABLISHED
tcp        0      0 localhost:50001         localhost:55798         SYN_RECV
tcp        0      0 localhost:55790         localhost:50001         ESTABLISHED
tcp        0      0 localhost:55796         localhost:50001         ESTABLISHED   

```

整理一下就是下面这样

<p align="center"><img src="/assets/img/tcp-backlog/pair.png"></p>

从上面可以看出,一共有**5**条连接对是`ESTABLISHED<->ESTABLISHED`连接, 但还有**4**条连接对是`SYN_RECV<->ESTABLISHED`连接, 这表示对**客户端**三次握手已经完成了,但对**服务器**还没有! 回顾一下`TCP`三次握手的过程,造成这种连接对原因只有可能是**服务器**将**客户端**最后发送的握手`ACK`被丢弃了!

还有一个问题,我明明设置的`backlog`的值是 **4**,可为什么还能建立**5**个连接 ?!


----------

### 去内核找原因

> 我实验用的机器内核是`4.4.0`

前面提到过**已完成连接队列**和**未完成连接队列**这两个概念, `Linux`有这两个队列吗 ? `Linux` 既有又没有! 说有是因为内核中可以得到两种连接各自的长度; 说没有是因为 `Linux`只有**已完成连接队列**实际存在, 而**未完成连接队列**只有长度的记录!

每一个`LISTEN`状态的套接字都有一个`struct inet_connection_sock`结构, 其中的`accept_queue`从名字上也可以看出就是已完成三次握手的子连接队列.只是这个结构里还记录了`半连接`请求的长度!

```c
struct inet_connection_sock {	
	// code omitted 
	struct request_sock_queue icsk_accept_queue;
    // code omitted
}

struct request_sock_queue {
	// code omitted
	atomic_t		qlen;                // 半连接sock的长度
	atomic_t		young;               //  

	struct request_sock	*rskq_accept_head;  // 已完成三次握手的sock队列头
	struct request_sock	*rskq_accept_tail;  // 已完成三次握手的sock队列尾
	// code omitted
}; 
```

需要注意的是这个 young, 一般情况下，它和 qlen 是共同变化的, 但有时不会，代码中的注释对这点有说明，即 1s 定时器超时都没有收到对端的 ACK。这个半连接就"成熟"了。

>Normally all the openreqs are young and become mature(i.e. converted to established socket) for first timeout.If synack was not acknowledged for 1 second, it means one of the following things: synack was lost, ack was lost, rtt is high or nobody planned to ack (i.e. synflood).
>

所以一般情况下连接建立时,服务端的变化过程是这样的:

 1. 收到`SYN`报文, `qlen`++,`young`++
 2. 收到`ACK`报文, 三次握手完成, 子连接加入`accept`队列, 半连接删除，`qlen`--,`young`--
 3. 用户使用`accept`,将连接从`accept`取出.


再来看内核收到`SYN`握手报文时的处理, 由于我关闭了`syncookie`,所以一旦满足了下面代码中的两个条件之一就会丢弃报文
```c

int tcp_conn_request(struct request_sock_ops *rsk_ops, 
		     const struct tcp_request_sock_ops *af_ops,
		     struct sock *sk, struct sk_buff *skb) 

	if ((net->ipv4.sysctl_tcp_syncookies == 2 ||
	     inet_csk_reqsk_queue_is_full(sk)) && !isn) {   // 条件1: 半连接数量 >= backlog
		want_cookie = tcp_syn_flood_action(sk, skb, rsk_ops->slab_name);
		if (!want_cookie)
			goto drop;
	} 

	if (sk_acceptq_is_full(sk) && inet_csk_reqsk_queue_young(sk) > 1) { // 条件2: 全连接sock > backlog 并且 半连接队列的young字段 > 1
		NET_INC_STATS(sock_net(sk), LINUX_MIB_LISTENOVERFLOWS);
		goto drop;
	} 

    // code omitted
```

"半连接队列的young字段 > 1" 表示网络很忙，有 SYNACK 丢失了(没有收到对端的第三次握手的 ACK)，但在我们的简单例子中，client 的 ACK 总是很及时的，所以这个条件不会满足，也就是说丢弃 SYN 报文的条件 2 只剩下**全连接sock > backlog** 

下面是收到`ACK`握手报文时的处理

```c
struct sock *tcp_v4_syn_recv_sock(const struct sock *sk, struct sk_buff *skb,
				  struct request_sock *req,
				  struct dst_entry *dst,
				  struct request_sock *req_unhash,
				  bool *own_req) 
{
     // code omitted
     if (sk_acceptq_is_full(sk))           //  全连接 > backlog, 就丢弃
	 	goto exit_overflow;

	 newsk = tcp_create_openreq_child(sk, req, skb); // 创建子套接字了
     // code omitted
}
```

所以这样就可以解释实验现象了!

 1. 前**4**个连接请求都可以顺利创建子连接, 全连接队列长度 = `backlog` = **4**, 半连接数目 = **0**
 2. 第**5**个连接请求, 由于`sk_acceptq_is_full`的判断条件是`>`而不是`>=`,所以依然可以建立全连接 
 3. 第**6-9**个连接请求到来时,由于半连接的数目还没有超过`backlog`,所以还是可以继续回复`SYNACK`,但收到`ACK`后已经不能再创建子套接字了,所以`TCP`状态依然为`SYN_RECV`.同时半连接的数目也增加到`backlog`.而对于客户端,它既然能收到`SYNACK`握手报文,因此它可以将`TCP`状态变为`ESTABLISHED`,
 4. 第**10**个请求到来时, 由于半连接的数目已经达到`backlog`,因此,这个`SYN`报文会被丢弃.


### 内核的问题

从以上的现象和分析中,我认为内核存在以下问题

 1. `accept`队列是否满的判断用`>=`比`>`更合适, 这样才能体现`backlog`的作用
 2. `accept`队列满了,就应该拒绝半连接了,因为即使半连接握手完成,也无法加入`accept`队列,否则就会出现`SYN_RECV--ESTABLISHED`这样状态的连接对!这样的连接是不能进行数据传输的!


问题`2`在16年的[补丁][5]中已经修改了! 所以如果你在更新版本的内核中进行相同的实验, 会发现客户端只能连接成功**5**次了，当然这也要先关闭`syncookie`

但问题`1`还没有修改！ 如果以后修改了，我也不会意外

(完)

### REF

[how-tcp-backlog-works-in-linux][6]


  [3]: http://pubs.opengroup.org/onlinepubs/9699919799/functions/listen.html
  [5]: https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/net/ipv4/tcp_input.c?id=5ea8ea2cb7f1d0db15762c9b0bb9e7330425a071
  [6]: http://veithen.io/2014/01/01/how-tcp-backlog-works-in-linux.html