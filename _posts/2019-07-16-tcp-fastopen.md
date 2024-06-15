---
layout    : post
title     : "TCP Fast Open(TFO)"
date      : 2019-07-16
lastupdate: 2019-07-16
categories: Network(Kernel)
---

<p align="center"><img src="/assets/img/tcp-fastopen/foc_use.png"></p>

### TCP Fast Open 的来源

网络的速度与两个因素相关：传输时延(transmission delay)和传播时延(propagation delay)。`transmission`是指将报文灌入网络电缆的时间，这是与带宽连接有关系的概念，比如千兆网络(网卡)比百兆网络(网卡)的`transmission delay`更小；

而`propagation delay`是指电信号在网络端到端的时延，它的大小只与端到端的距离有关(电信号以光速传播)，TCP 中的往返时间`Round Trip Time`也就是与这个时延相关。

然而，TCP 并不能减小上面任何一种 delay，它能做的只能是想办法让端到端的通信更有效率。什么意思呢？我们知道当前网络上的 TCP 流量大部分是短连接(short TCP conversation)，比如浏览 http 网页。这种连接的特点是：连接持续时间不长，除去三次握手和四次挥手的控制报文交互，它们之间的数据报文并不多。这就使得三次握手报文这个不带数据的控制报文显得有点浪费了(白白花费`propagation delay`)。

尽管 [RFC793]((https://tools.ietf.org/html/rfc793)) 并没有禁止 SYN 报文携带数据，但所有的 TCP 实现默认都不会使用。原因是这不太安全，站在 Server 的角度，收到这样一个 SYN 报文，但这个时候 TCP 握手还没完成呢，对端真的可信吗？说不定是一个伪造源端的 TCP 报文(报文中的源 IP 并不是自己控制的)，稳妥起见，这个数据报文还是等握手完成之后再上送给应用吧。

但使用者希望追求极致得到效率，在 SYN 报文中就带上用户数据。

这就是 TCP Fast Open 的来源，它允许在第一个握手的 SYN 报文中携带数据，如此以来，短连接便可以节省一次来回的`propagation delay`

### TCP Fast Open 的原理

TFO 的基本思想用一句话概括就是："一回生，二回熟"。站在 Server 的角度，如果它开启了 TFO 功能的话，它会为首次发起连接的 Clinet (以源IP区分)发放一个专属的通信证(根据 IP 地址生成的 Cookie)，在这之后，同一个 Clinet 如果还要向 Server 发起 TCP 连接，在 SYN 报文中带上 cookie 和用户数据。Server 验证这个 Cookie 通过后，就可以直接将报文上送给应用.

#### 首次连接建立过程

首次 TCP 连接的建立过程如下图所示(来自[RFC7413](https://tools.ietf.org/html/rfc7413)), Clinet 向 Server 发送的 SYN 报文中带上了 Cookie 为空的 TCP 选项，表示自己希望使用 TFO 功能，但还没有通信证(Cookie)，因此需要请求。

开启了 TFO 功能的 Server 在收到该 SYN 报文后，会生成 Cookie，通过 SYNACK 报文的选项字段传回。

Clinet 收到 SYNACK 报文后，便会缓存下该 Cookie。

```
 Requesting Fast Open Cookie in connection 1:

   TCP A (Client)                                      TCP B (Server)
   ______________                                      ______________
   CLOSED                                                      LISTEN

   #1 SYN-SENT       ----- <SYN,CookieOpt=NIL>  ---------->  SYN-RCVD

   #2 ESTABLISHED    <---- <SYN,ACK,CookieOpt=C> ----------  SYN-RCVD
   (caches cookie C)
```

<p align="center"><img src="/assets/img/tcp-fastopen/cookie_request.PNG"></p>
<p align="center">Figure.1 Clinet 发送带 Cookie 请求的 SYN 报文</p>

<p align="center"><img src="/assets/img/tcp-fastopen/cookie_responde.PNG"></p>
<p align="center">Figure.2 Server 回复带 Cookie 的 SYNACK 报文</p>

##### 后续连接建立过程

当 Clinet 后续再发起连接时，由于已经它已经有了 Cookie，因此它可以在第一个 SYN 报文时就携带数据(DATA_A),

Server 在收到该 SYN 报文后，如果 Cookie 验证通过，便会将数据上送给应用程序，即使现在三次握手还没有完成，TCP 套接字还处于(SYN_RCVD)状态

```
 Performing TCP Fast Open in connection 2:

   TCP A (Client)                                      TCP B (Server)
   ______________                                      ______________
   CLOSED                                                      LISTEN

   #1 SYN-SENT       ----- <SYN=x,CookieOpt=C,DATA_A> ---->  SYN-RCVD

   #2 ESTABLISHED    <---- <SYN=y,ACK=x+len(DATA_A)+1> ----  SYN-RCVD

   #3 ESTABLISHED    <---- <ACK=x+len(DATA_A)+1,DATA_B>----  SYN-RCVD

   #4 ESTABLISHED    ----- <ACK=y+1>--------------------> ESTABLISHED

   #5 ESTABLISHED    --- <ACK=y+len(DATA_B)+1>----------> ESTABLISHED
```

<p align="center"><img src="/assets/img/tcp-fastopen/SYNwithCookie.PNG"></p>
<p align="center">Figure.3 Clinet 发送带 Cookie 的 SYNA 报文</p>


#### Cookie 的格式

Cookie 通过 TCP 的选项(Kind = 34)在 TCP 双方之间交互，其格式如下。它的值由 Server 根据 <ClinetIP、ServerIP> 生成。注意，Cookie 与 TCP 端口号无关，即使应用程序不同，只要 Client 和 Server 使用的 IP 不变，两台主机上的 TCP 程序就可以复用一个 Cookie，换句话说，这个 Cookie 是主机粒度的。 

```
                                   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
                                   |      Kind     |    Length     |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |                                                               |
   ~                            Cookie                             ~
   |                                                               |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

   Kind            1 byte: value = 34
   Length          1 byte: range 6 to 18 (bytes); limited by
                           remaining space in the options field.
                           The number MUST be even.
   Cookie          0, or 4 to 16 bytes (Length - 2)
```

### 在 Linux 中使用 TFO

#### 开启系统 TFO 功能 

TFO 功能需要在 TCP 通信的双方都启用时才会生效，内核的 TFO 功能在 3.6 (Clinet-Side) 和 3.7 (Server-Side)被集成进内核。支持 TFO 的内核可以通过设置`/proc/sys/net/ipv4/tcp_fastopen`控制其 Clinet 端和 Server 端的 TFO 开关。

```
# 开启 Clinet-Side TFO
> echo 1 > /proc/sys/net/ipv4/tcp_fastopen

# 开启 Server-Side TFO
> echo 2 > /proc/sys/net/ipv4/tcp_fastopen

# 开启 Server-Side & Clinet-Side TFO
> echo 3 > /proc/sys/net/ipv4/tcp_fastopen
```

#### User-Space API

对于使用 TFO 的应用程序来说，并不需要关心 Cookie 的缓存、发送，这些工作都由内核完成，应用程序唯一需要做的就是**告诉**内核自己需要使用 TFO.

**Server**唯一需要增加的步骤是在`listen()`之前，设置 `TCP_FASTOPEN` 的 socket 选项，选项值表示可以进行的 TFO 请求数量的最大值。
```c
    sfd = socket(AF_INET, SOCK_STREAM, 0);   // Create socket

    bind(sfd, ...);                          // Bind to well known address
    
    int qlen = 5;                            // Value to be chosen by application
    setsockopt(sfd, SOL_TCP, TCP_FASTOPEN, &qlen, sizeof(qlen));
    
    listen(sfd, ...);                        // Mark socket to receive connections

    cfd = accept(sfd, NULL, 0);              // Accept connection on new socket

    // read and write data on connected socket cfd

    close(cfd);
```
 
**Clinet**需要做的是将原本使用`connect()`和`send()`的地方，替换成`sendto()`或者`sendmsg()`，并带上`MSG_FASTOPEN`标识。 
```c 
   sfd = socket(AF_INET, SOCK_STREAM, 0);
    
   sendto(sfd, data, data_len, MSG_FASTOPEN, 
                (struct sockaddr *) &server_addr, addr_len);
    // Replaces connect() + send()/write()
    
    // read and write further data on connected socket sfd

    close(sfd);
```

### Linux 中的 TFO 实现

所选代码示例为 **4.4.0** 版本

#### Clinet 发起 TFO 请求

和普通 SYN 报文的构建过程不同，使用 TFO 时，Clinet 使用`tcp_send_syn_data()`组装构建 SYN 报文。

它会从内核 metrics 框架中查询是有目标 Server 对应的 Cookie，若有，则直接填充到 SYN 报文中，若没有，则填充一个长度为 0 的Cookie 选项。 

```
tcp_sendmsg
  |
  |-- tcp_sendmsg_fastopen
      |
      |-- tcp_connect
          |
          |-- tcp_send_syn_data
          
/* Build and send a SYN with data and (cached) Fast Open cookie. However,
 * queue a data-only packet after the regular SYN, such that regular SYNs
 * are retransmitted on timeouts. Also if the remote SYN-ACK acknowledges
 * only the SYN sequence, the data are retransmitted in the first ACK.
 * If cookie is not cached or other error occurs, falls back to send a
 * regular SYN with Fast Open cookie request option.
 */
static int tcp_send_syn_data(struct sock *sk, struct sk_buff *syn)
{
	struct tcp_sock *tp = tcp_sk(sk);
	struct tcp_fastopen_request *fo = tp->fastopen_req;
	int syn_loss = 0, space, err = 0;
	unsigned long last_syn_loss = 0;
	struct sk_buff *syn_data;

	tp->rx_opt.mss_clamp = tp->advmss;  /* If MSS is not cached */
	tcp_fastopen_cache_get(sk, &tp->rx_opt.mss_clamp, &fo->cookie,   // 从 cache 中获取是否已有 Cookie，放入 fo->cookie
			       &syn_loss, &last_syn_loss);

    // code omitted 
}                   
```

#### Server 收到带 TFO Cookie 请求的 SYN 报文

Server 收到带 TFO Cookie 请求的 SYN 报文, 调用`tcp_try_fastopen()`，这里并不会直接创建 child 连接，原因是收到的 SYN 只带了 Cookie 请求，Server 随后会通过`tcp_fastopen_cookie_gen()`创建有效的 Cookie，存入`valid_foc`，最后用`foc`带出去后组装 SYNACK 发送出去.

```c
tcp_conn_request
    |
    |-- tcp_try_fastopen
    |
    |-- af_ops->send_synack(fastopen_sk, dst, &fl, req, &foc, false);
    
struct sock *tcp_try_fastopen(struct sock *sk, struct sk_buff *skb, // sk 是 lisnter
			      struct request_sock *req,
			      struct tcp_fastopen_cookie *foc,
			      struct dst_entry *dst)
{
	// code omitted
	if (foc->len >= 0 &&  /* Client presents or requests a cookie */
	    tcp_fastopen_cookie_gen(req, skb, &valid_foc) &&                 
	    foc->len == TCP_FASTOPEN_COOKIE_SIZE &&
	    foc->len == valid_foc.len &&
	    !memcmp(foc->val, valid_foc.val, foc->len)) {
		// Cookie 有效, 创建子连接
fastopen:
		child = tcp_fastopen_create_child(sk, skb, dst, req);
		if (child) {
			foc->len = -1;
			NET_INC_STATS_BH(sock_net(sk),
					 LINUX_MIB_TCPFASTOPENPASSIVE);
			return child;
		}
		NET_INC_STATS_BH(sock_net(sk), LINUX_MIB_TCPFASTOPENPASSIVEFAIL);
	} else if (foc->len > 0) /* Client presents an invalid cookie */
		NET_INC_STATS_BH(sock_net(sk), LINUX_MIB_TCPFASTOPENPASSIVEFAIL);

	valid_foc.exp = foc->exp;
	*foc = valid_foc;
	return NULL;
}
```

#### Clinet 收到带 TFO Cookie 的 SYNACK 报文


```c 
tcp_v4_do_rcv
    |
    |-- tcp_rcv_state_process
        |
        |-- tcp_rcv_synsent_state_process
            |
            |-- tcp_rcv_fastopen_synack
            
static bool tcp_rcv_fastopen_synack(struct sock *sk, struct sk_buff *synack,
				    struct tcp_fastopen_cookie *cookie)
{
    // code omitted
    
    /* 将 SYNACK 报文中的 Cookie 缓存起来(保存到 metrics 框架) */
	tcp_fastopen_cache_set(sk, mss, cookie, syn_drop, try_exp);
   
    // code omitted 
}            
```

#### Server 收到 Clinet 后续发起的新连接

Clinet 后续向 Server 发送的 SYN 请求会携带 Cookie，Server 收到后回立即创建子连接(设置为 SYN-RCVD 状态)，之后收到 Clinet 的 ACK 后再更改为 ESTABLISHED 状态。 

```c
tcp_conn_request
    |
    |-- child = tcp_try_fastopen

tcp_conn_request    
{  
    if (!want_cookie) {
		tcp_reqsk_record_syn(sk, req, skb);
		fastopen_sk = tcp_try_fastopen(sk, skb, req, &foc, dst);  
	}
	if (fastopen_sk) {
		af_ops->send_synack(fastopen_sk, dst, &fl, req,
				    &foc, false);
		/* Add the child socket directly into the accept queue */
		inet_csk_reqsk_queue_add(sk, req, fastopen_sk);
		sk->sk_data_ready(sk);
        // code omitted
    }
}
 
```

### REF 

- [TCP Fast Open: expediting web services](https://lwn.net/Articles/508865/)
- [RFC7413 TCP Fast Open](https://tools.ietf.org/html/rfc7413)