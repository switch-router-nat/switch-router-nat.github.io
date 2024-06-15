---
layout    : post
title     : "Linux内核中reuseport的演进"
date      : 2020-09-30
lastupdate: 2020-09-30
categories: Network(Kernel)
---

<p align="center"><img src="/assets/img/public/tcp.png"></p>

`SO_REUSEPORT`选项在Linux 3.9被引入内核，在这之前也有一个很像的选项`SO_REUSEADDR`。

如果你不太清楚这两者的区别和联系，可以先阅读[How do SO_REUSEADDR and SO_REUSEPORT differ?](https://stackoverflow.com/questions/14388706/how-do-so-reuseaddr-and-so-reuseport-differ)。

如果不想读，那么下面这一节算是为懒人准备的。

### SO_REUSEADDR 与 SO_REUSEPORT 是什么?

TCP/UDP用`五元组`唯一标识一个连接。**任何时候**，两条连接的五元组都不能完全相同，否则当收到一个报文时，协议栈没办法判断它是属于哪个连接的。

```bash
五元组
{<protocol>, <src addr>, <src port>, <dest addr>, <dest port>}
```

五元组里，`protocol`在创建socket时确定，`<src addr>`和`<src port>`在`bind()`时确定，`<dest addr>`和`<dest port>`在`connect()`时确定。

当然，`bind()`和`connect()`在一些时候并不需要显式使用，不过这不在本文的讨论范围里。

那么，如果对socket设置了`SO_REUSEADDR`和`SO_REUSEPORT`选项，它们什么时候起作用呢？ 答案是`bind()`，也就在确定`<src addr>`和`<src port>`时。

不同操作系统内核对待`SO_REUSEADDR`和`SO_REUSEPORT`的行为有少许差异，但它们都源自 BSD。因此，接下来就以BSD的实现为标准进行说明。

#### SO_REUSEADDR

假设我现在需要`bind()`将`socketA`绑定到`A:X`，将`socketB`绑定到`B:Y`(不考虑`X=0`或者`Y=0`，因为`0`表示让内核自动分配端口，一定不会冲突)。

如果`X!=Y`，那么无论`A`和`B`的关系如何，两个`bind()`都会成功。但如果`X==Y`，那么结果会是下面这样:

```
SO_REUSEADDR       socketA        socketB       Result
---------------------------------------------------------------------
  ON/OFF       192.168.0.1:21   192.168.0.1:21    Error (EADDRINUSE)
  ON/OFF       192.168.0.1:21      10.0.0.1:21    OK
  ON/OFF          10.0.0.1:21   192.168.0.1:21    OK
   OFF             0.0.0.0:21   192.168.1.0:21    Error (EADDRINUSE)
   OFF         192.168.1.0:21       0.0.0.0:21    Error (EADDRINUSE)
   ON              0.0.0.0:21   192.168.1.0:21    OK
   ON          192.168.1.0:21       0.0.0.0:21    OK
  ON/OFF           0.0.0.0:21       0.0.0.0:21    Error (EADDRINUSE)
```

第一列表示是否设置`SO_REUSEADDR`<sup>`注`</sup>，最后一列表示**<mark>后绑定</mark>**的socket是否能绑定成功。

<sup>`注`</sup>：这里设置的对象是指**<mark>后绑定</mark>**的socket(也就是说不关心前一个是否设置)

可以看出，BSD的实现中`SO_REUSEADDR`可以让**<mark>一个使用通配地址(0.0.0.0)，一个使用指定地址(192.168.1.0)的socket同时绑定成功</mark>**。

`SO_REUSEADDR`还有一种应用情景：假设`socketA`绑定到`A:X`，完成通信后使用`close()` 进入`TIME_WAIT`状态，

此时，如果`socketB`也去绑定`A:X`，此时同样会得到`EADDRINUSE`错误，但如果`socketB`设置了`SO_REUSEADDR`，则可以绑定成功。

#### SO_REUSEPORT

如果理解了`SO_REUSEADDR`，那么`SO_REUSEPORT`就很好理解了，**<mark>它可以让两个socket可以绑定完全相同的地址端口/mark>**。
```
SO_REUSEPORT       socketA        socketB       Result
---------------------------------------------------------------------
    ON         192.168.0.1:21   192.168.0.1:21    OK
```

**<mark>注意，以上的结果都是BSD的结果，Linux内核有一些不一样的地方</mark>**，具体表现为

- 3.9版本支持`SO_REUSEPORT`，作为Server的TCP Socket一旦绑定到了具体的端口，启动了LISTEN，即使它之前设置过`SO_REUSEADDR`, 也不会生效。这一点Linux比BSD更加严格
```
SO_REUSEADDR       socketA        socketB       Result
---------------------------------------------------------------------
    ON/OFF      192.168.0.1:21   0.0.0.0:21    Error (EADDRINUSE)
```
- 3.9版本之前,作为Client的Socket，`SO_REUSEADDR`选项具有BSD中的`SO_REUSEPORT`的效果。这一点Linux又比BSD更加宽松。
```
SO_REUSEADDR      socketA            socketB           Result
---------------------------------------------------------------------
    ON        192.168.0.2:55555   192.168.0.2:55555      OK
```

### Linux中reuseport的演进

#### Linux < 3.9 

下面看看具体是怎么做的:

内核socket使用`skc_reuse`字段表示是否设置了`SO_REUSEADDR`
```c
 struct sock_common {
 	/* omitted */
    unsigned char		skc_reuse;
    /* omitted */
}

int sock_setsockopt(struct socket *sock, int level, int optname,...
{
    ......
    case SO_REUSEADDR:
 	sk->sk_reuse = (valbool ? SK_CAN_REUSE : SK_NO_REUSE);
 	break;
}
```

`inet_bind_bucket`表示一个绑定的端口。
```c
struct inet_bind_bucket {
    /* omitted */
	unsigned short		port;
	signed short		fastreuse;
	int			num_owners;
	struct hlist_node	node;
	struct hlist_head	owners;
};
```
上面结构中的`fastreuse`表示该端口是否支持共享，所有共享该端口的socket挂到`owner`成员上。在用户使用`bind()`时，内核使用**TCP**:`inet_csk_get_port()`,**UDP**:`udp_v4_get_port()`来绑定端口。
```c
/* inet_connection_Sock.c: inet_csk_get_port() */
tb_found:
	if (!hlist_empty(&tb->owners)) {
        ......
		if (tb->fastreuse > 0 &&
		    sk->sk_reuse && sk->sk_state != TCP_LISTEN &&
		    smallest_size == -1) {
			goto success;
```
所以，当该端口支持共享，且socket也设置了`SO_REUSEADDR`并且不为`LISTEN`状态时，此次`bind()`可以成功。

#### 3.9 =< Linux < 4.5 

`3.9`版本内核增加了对`SO_REUSEPORT`的支持，`listener`可以绑定到相同的`<IP:Port>`了。这个时候，当Server收到Client发送的SYN报文时，会选择其中一个socket进行响应.

[图]

具体到实现，`3.9`版本扩展了`sock_common`，将原来记录`skc_reuse`进行了拆分.

```diff
struct sock_common {
 	unsigned short		skc_family;
 	volatile unsigned char	skc_state;
-	unsigned char		skc_reuse;
+	unsigned char		skc_reuse:4;
+	unsigned char		skc_reuseport:4;


@@ int sock_setsockopt(struct socket *sock, int level, int optname,
 	case SO_REUSEADDR:
 		sk->sk_reuse = (valbool ? SK_CAN_REUSE : SK_NO_REUSE);
 		break;
+	case SO_REUSEPORT:
+		sk->sk_reuseport = valbool;
+		break;

```

然后对`inet_bind_bucket`也相应进行了扩展
```diff
struct inet_bind_bucket {
 	/* omitted */
 	unsigned short		port;
-	signed short		fastreuse;
+	signed char		fastreuse;
+	signed char		fastreuseport;
+	kuid_t			fastuid;
```

而在绑定端口时，增加了一个队reuseport的通过条件

```diff
/* inet_connection_sock.c: inet_csk_get_port() */
tb_found:
 		if (sk->sk_reuse == SK_FORCE_REUSE)
 			goto success;
-		if (tb->fastreuse > 0 &&
-		    sk->sk_reuse && sk->sk_state != TCP_LISTEN &&
+		if (((tb->fastreuse > 0 &&
+		      sk->sk_reuse && sk->sk_state != TCP_LISTEN) ||
+		     (tb->fastreuseport > 0 &&
+		      sk->sk_reuseport && uid_eq(tb->fastuid, uid))) 
             && smallest_size == -1) {
               goto success;
```

而当Client的SYN报文到达时，Server会首先根据本地端口(SYN报文的`<dport>`)计算出一条hash冲突链，然后遍历该链表上的所有Socket，根据四元组匹配程度进行打分;如果使能了reuseport，那么可能有多个Socket都将拿到最高分，此时内核将随机选择一个进行后续处理。

```c
/* inet_hashtables.c  */
struct sock *__inet_lookup_listener(struct......)
{
	struct sock *sk, *result;
	unsigned int hash = inet_lhashfn(net, hnum);
	struct inet_listen_hashbucket *ilb = &hashinfo->listening_hash[hash]; // 根据本地端口找到hash冲突链
    /* code omitted */
	result = NULL;
	hiscore = 0;
	sk_nulls_for_each_rcu(sk, node, &ilb->head) {
		score = compute_score(sk, net, hnum, daddr, dif); // 根据匹配程度进行打分
		if (score > hiscore) {
			result = sk;
			hiscore = score;
			reuseport = sk->sk_reuseport;
			if (reuseport) {
				phash = inet_ehashfn(net, daddr, hnum,
						     saddr, sport);
				matches = 1;                             // 如果是reuseport 则累计多少个socket满足
			}
		} else if (score == hiscore && reuseport) {
			matches++;
			if (reciprocal_scale(phash, matches) == 0)
				result = sk;
			phash = next_pseudo_random32(phash);
		}
	}
	/*
	 * if the nulls value we got at the end of this lookup is
	 * not the expected one, we must restart lookup.
	 * We probably met an item that was moved to another chain.
	 */
	return result;
}
```

举个栗子，假设内核有4条listening socket的hash冲突链，然后用户建立了4个Server：A、B、C、D，监听的地址和端口如下图所示，A和B使能了`SO_REUSEPORT`。冲突链是以端口为Key的，因此A、B、D会挂到同一条冲突链上。如果此时收到对端一个SYN报文<192.168.10.1, 21>,那么内核会遍历`listening_hash[0]`，为上面的7个socket进行打分，而由于B监听的是精确的地址，所以B的得分会比A高，内核最终选择出一个SocketB进行后续处理。

<p align="center"><img src="/assets/img/reuseport/3_9-4_5.png"></p>

#### 4.5 < Linux 

从上面的例子可以看出，当收到SYN报文时，内核一定会遍历一条完整hash冲突链，为每一个socket进行打分，这稍微有些多余。因此，在4.5版本中，内核引入了`reuseport groups`，它将绑定到同一个IP和Port，并且设置了`SO_REUSEPORT`选项的socket组织到一个`group`内部。

<p align="center"><img src="/assets/img/reuseport/4_5.png"></p>

```diff
--- a/include/net/sock.h
+++ b/include/net/sock.h
@@ -318,6 +318,7 @@ struct cg_proto;
   *	@sk_error_report: callback to indicate errors (e.g. %MSG_ERRQUEUE)
   *	@sk_backlog_rcv: callback to process the backlog
   *	@sk_destruct: called at sock freeing time, i.e. when all refcnt == 0
+  *	@sk_reuseport_cb: reuseport group container
  */
 struct sock {
 	/*
@@ -453,6 +454,7 @@ struct sock {
 	int			(*sk_backlog_rcv)(struct sock *sk,
 						  struct sk_buff *skb);
 	void                    (*sk_destruct)(struct sock *sk);
+	struct sock_reuseport __rcu	*sk_reuseport_cb;
 };
```

这个特性在4.5版本只支持UDP,而在4.6版本开始支持TCP([patch](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/net/ipv4/inet_hashtables.c?id=c125e80b88687b25b321795457309eaaee4bf270))。这样在查找listen socket时，内核将不用再遍历整个冲突链，而是在找到一个合格的socket时，如果它设置了`SO_REUSEPORT`,就直接找到它所属的`reuseport group`,从中选择一个进行后续处理.

```diff
@@ -215,6 +217,7 @@ struct sock *__inet_lookup_listener(struct net *net,
 	unsigned int hash = inet_lhashfn(net, hnum);
 	struct inet_listen_hashbucket *ilb = &hashinfo->listening_hash[hash];
 	int score, hiscore, matches = 0, reuseport = 0;
+	bool select_ok = true;
 	u32 phash = 0;
 
 	rcu_read_lock();
@@ -230,6 +233,15 @@ begin:
 			if (reuseport) {
 				phash = inet_ehashfn(net, daddr, hnum,
 						     saddr, sport);
+				if (select_ok) {
+					struct sock *sk2;
+					sk2 = reuseport_select_sock(sk, phash,
+								    skb, doff);
+					if (sk2) {
+						result = sk2;
+						goto found;
+					}
+				}
 				matches = 1;
 			}
 		}
```







