---
layout    : post
title     : "内核 strparser 是如何工作的"
date      : 2020-07-25
lastupdate: 2020-07-25
categories: Network(Kernel)
---

### strparser 是怎么工作的

**strparser**是 Linux 内核在 4.9 版本引入的 feature (https://lwn.net/Articles/703116/)。它允许用户在内核层面拦截送往 TCP 套接字的报文并做自定义的处理。处理的地方可以是内核模块，也可以是 eBPF 程序。

- 内核模块处理截获报文的例子：KTLS（https://github.com/ktls/af_ktls）

KTLS 这个 feature 已经进入内核代码主线了，它的设计思想是让 TLS 需要的加解密操作就在内核层面就完成，而不必拷贝之后在用户态做，根据论文 https://netdevconf.info/1.2/papers/ktls.pdf的分析结果，这样做可以减少 7% 的CPU 的消耗和 10% 的传输时延。

<p align="center"><img src="/assets/img/strparser/KTLS.PNG"></p>

- eBPF程序处理截获报文的例子：psock 

psock 使用 strpaser，将数据包的控制权转移到 eBPF 处理程序，用户可以在 eBPF 程序里完成网络报文的重定向，一个例子是(https://blog.csdn.net/dog250/article/details/103629054)，sockmap 建立在 psock 之上，而 psock 的底座则是 strparser.

#### strparser 的工作原理

##### 核心数据结构

```c
struct strparser {
	struct sock *sk;
	// code omitted ....
	struct strp_callbacks cb;
};
```

**struct strparser** 是 strparser 框架的核心数据结构，它绑定(attach)一个 TCP sock 结构 **sk** 和一组回调函数 **cb**。

```c
strp_init(struct strparser *strp, struct sock *csk, struct strp_callbacks *cb)
```

*strp_init()* 完成 **struct strparser** 的初始化，它的参数就是要绑定的 TCP 连接和使用者设置的回调函数。strparser 框架在合适的时候调用这些回调函数。

回调函数一共有以下六个：

```c
struct strp_callbacks {
	int (*parse_msg)(struct strparser *strp, struct sk_buff *skb);
	void (*rcv_msg)(struct strparser *strp, struct sk_buff *skb); 
	int (*read_sock_done)(struct strparser *strp, int err)
	void (*abort_parser)(struct strparser *strp, int err);
	void (*lock)(struct strparser *strp);
	void (*unlock)(struct strparser *strp);
};
```

其中

```c
int (*parse_msg)(struct strparser *strp, struct sk_buff *skb);
```

*parse_msg()* 在 strpaser 收到报文时被框架调用。它用于从报文中提取下一个应用层消息(message)的长度。一个 TCP 报文里可能不止一个应用层消息，而 *parse_msg()* 就是提供给使用者去识别各个消息的手段。

![image-20200803165705523](C:\Users\liuyacan\AppData\Roaming\Typora\typora-user-images\image-20200803165705523.png)

当然，如果应用在一个 TCP 报文里只放置一个消息，则它可以实现为直接返回 skb->len 即可

```c
void (*rcv_msg)(struct strparser *strp, struct sk_buff *skb);
```
*rcv_msg()* 在消息被 parse 之后调用，用于将报文交给上层使用者。

##### strpaser 截获报文

正常情况下，内核 TCP 层处理报文后，会调用 *sock->sk_data_ready(sk)* , 它的默认动作是 wake up 一个用户态进程.

```c
void tcp_data_ready(struct sock *sk)
{
	const struct tcp_sock *tp = tcp_sk(sk);
	// code omitted

	sk->sk_data_ready(sk);
}
```

我们期望报文能进入 **strpaser** ，但报文显然不会平白无故地地进入 **strpaser** ，因此，我们需要在报文的上送路径上动一些手脚：替换掉 *sk->sk_data_ready* 函数

 KTLS 的例子中，在做好备份后， *tls_data_ready()* 替换被赋值到 *sk->sk_data_ready* 。

```c
static int tls_bind(struct socket *sock, struct sockaddr *uaddr, int addr_len){
    // code omitted
    tsk->saved_sk_data_ready = tsk->socket->sk->sk_data_ready;
	tsk->saved_sk_write_space = tsk->socket->sk->sk_write_space;sk_write_space
	tsk->socket->sk->sk_data_ready = tls_data_ready; 
	tsk->socket->sk->sk_write_space = tls_write_space;
	tsk->socket->sk->sk_user_data = tsk;     
    // code omitted
}
```

同样地，在 psock 的例子中， *sk_psock_strp_data_ready()* 被赋值到 *sk->sk_data_ready*

```c
void sk_psock_start_strp(struct sock *sk, struct sk_psock *psock)
{
	struct sk_psock_parser *parser = &psock->parser;
    // code omitted
	parser->saved_data_ready = sk->sk_data_ready;
	sk->sk_data_ready = sk_psock_strp_data_ready;
	sk->sk_write_space = sk_psock_write_space;
	parser->enabled = true;
}
```

替换之后，当有 TCP 报文准备上送时，用户定义的 sk->sk_data_ready 函数就会被调用，在该函数中，KTLS/psock 需要调用框架函数*strp_data_ready()* 将报文转交给 **strpaser** 框架。

对 KTLS 

```c
static void tls_data_ready(struct sock *sk)
{
	struct tls_context *tls_ctx = tls_get_ctx(sk);
	struct tls_sw_context_rx *ctx = tls_sw_ctx_rx(tls_ctx);

	strp_data_ready(&ctx->strp);
}
```

对 psock 

```c
static void sk_psock_strp_data_ready(struct sock *sk)
{
	struct sk_psock *psock;

	rcu_read_lock();
	psock = sk_psock(sk);
	if (likely(psock)) {
		write_lock_bh(&sk->sk_callback_lock);
		strp_data_ready(&psock->parser.strp);
		write_unlock_bh(&sk->sk_callback_lock);
	}
	rcu_read_unlock();
}
```

##### strpaser 处理报文 

**strpaser** 框架拿到报文之后，通常会依次调用用户设置的 *parse_msg* 和 *rcv_msg* 回调函数，用户在回调函数里用来决定报文应该何去何从

```
strp_data_ready
  |- strp_read_sock
    |- tcp_read_sock
       |- strp_recv
         |- __strp_recv
           |- strp->cb.parse_msg(strp, head)
           ...
           |- strp->cb.rcv_msg(strp, head);
```

比如对 KTLS,  就是将报文上送给应用层(AF_KTLS socket)

```c
static void tls_queue(struct strparser *strp, struct sk_buff *skb)
{
	struct tls_sock *tsk;
	
    // code omitted 
	tsk = strp->sk->sk_user_data;
	// code omitted 
	
	ret = sock_queue_rcv_skb((struct sock *)tsk, skb);
	// code omitted 
}

```

而对于 psock, 则是运行 eBPF 程序，得到动作(verdict)。

```c
static void sk_psock_strp_read(struct strparser *strp, struct sk_buff *skb)
{
	struct sk_psock *psock = sk_psock_from_strp(strp);
	struct bpf_prog *prog;
	int ret = __SK_DROP;

	rcu_read_lock();
	prog = READ_ONCE(psock->progs.skb_verdict);
	if (likely(prog)) {
		skb_orphan(skb);
		tcp_skb_bpf_redirect_clear(skb);
		ret = sk_psock_bpf_run(psock, prog, skb); // if we rdir , return SK_PASS
		ret = sk_psock_map_verd(ret, tcp_skb_bpf_redirect_fetch(skb));
	}
	rcu_read_unlock();
	sk_psock_verdict_apply(psock, skb, ret);
}
```

### 总结

**strpaser** 是一个框架，它本身规定限定如何处理报文，而只是在内核层面提供给了用户一个提前处理 TCP 报文的时机和一组回调函数，用户通过不同的回调函数可以实现不同的逻辑。