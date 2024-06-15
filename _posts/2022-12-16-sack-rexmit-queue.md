---
layout    : post
title     : "SACK与内核TCP重传队列"
date      : 2022-12-16
lastupdate: 2022-12-16
categories: Network(Kernel)
---

<p align="center"><img src="/assets/img/public/tcp.png"></p>

内核 TCP 重传队列并不是保存重传过的报文的队列，而是保存着尚未被对端确认的 `sk_buff` (也就是可能会被重传的报文), 

该队列以红黑树的形式存放在`struct sock` 结构中

```c
struct sock { 
    ...
    union {
		struct sk_buff	*sk_send_head;
		struct rb_root	tcp_rtx_queue;   // 红黑树的根
	};
	...
}
```

队列中(树上)的`sk_buff`按`seq`序号排列, 比如此刻假设 SND_UNA 为 101, 那么队首元素就为起始序号 101 的`sk_buff`.

`sk_buff`从队列中取出的条件是被**<mark>完全应答</mark>**

这意味着即使从某个 ack 报文的 SACK block 获知其中一段数据被对端接收, 发送方也不能从队列中将这段数据对应的 `sk_buff` 取出. 

取而代之的是, 会在这些`sk_buff`上进行一些标记.

TCP 连接的`sk_buff`的 cb 区域存放的是`struct tcp_skb_cb` 结构, 它里面有一个 8bit 的bitmap变量`sacked`专门保存 SACK 相关标记.

```
#define TCP_SKB_CB(__skb)	((struct tcp_skb_cb *)&((__skb)->cb[0]))

struct tcp_skb_cb {
	...
	__u8		sacked;					/* State flags for SACK.	*/
#define TCPCB_SACKED_ACKED	0x01		/* SKB ACK'd by a SACK block */
#define TCPCB_SACKED_RETRANS	0x02	/* SKB retransmitted */
#define TCPCB_LOST		0x04			/* SKB is lost */
    ...
```

这几个标记的意义为
`SACKED(S)` : 该`sk_buff`在某个 SACK block 中被应答
`RETRANS(R)`: 该`sk_buff`被重传过
`LOST(L)`   : 该`sk_buff`确认已经丢失

将这几个标记组合起来, 各自代表的意义在内核代码中也有注释解释.

```
* Tag  InFlight	Description
 * 0	1		- orig segment is in flight.
 * S	0		- nothing flies, orig reached receiver.
 * L	0		- nothing flies, orig lost by net.
 * R	2		- both orig and retransmit are in flight.
 * L|R	1		- orig is lost, retransmit is in flight.
 * S|R  1		- orig reached receiver, retrans is still in flight.
```

另外, 内核在`struct tcp_sock`上有几个变量记录着重传队列上一些报文的数量 
```c
struct tcp_sock {
	...
	u32	lost_out;	/* Lost packets	  */
	u32	sacked_out;	/* SACK'd packets */
	...
	u32	packets_out;	/* Packets which are "in flight" */
	u32	retrans_out;	/* Retransmitted packets out	 */
}
```
`lost_out` : 本端正处于丢失状态的报文数目
`sacked_out`: 本端正处于 SACK 已应答状态的报文数目
`packets_out`: 本端认为在途的状态的报文数目(只要没被完全确认, 就视为在途)
`retrans_out`: 本端眼中正在被重传的报文数目.

以上的解释有些枯燥, 还是举一个实际例子说吧.

sender 向 receiver 发送5个长度为1000字节的报文, 其中第2个和第4个报文丢失. receiver 向 sender 回复的 5 个ack中, 有两个带上了 SACK block, 本末附录贴出了对应的 packetdrill 脚本.

整个收发情况如下图所示, 括号中的数字表示 SACK block

<p align="center"><img src="/assets/img/sack-rexmit-queue/pic1.png"></p>

抓包结果是

<p align="center"><img src="/assets/img/sack-rexmit-queue/pic2.png"></p>

站在 sender 的视角, 逐个分析每个 ack 到达时的变化.

##### 1st ack 到达前

由于发送了5个报文, 因此在 1st ack 到达之前, 重传队列上存在5个`sk_buff`, 且它们的标记均未设置. 此时`tcp_sock` 上的状态如下
```
tp->retrans_out = 0 
tp->sacked_out = 0  
tp->lost_out = 0 
tp->packets_out = 5   // [1:1001] [1001:2001] [2001:3001] [3001:4001] [4001:5001]
```

##### 1st ack 到达

> 192.0.2.1.58393 > 192.168.165.149.9996: Flags [.], ack 1001, win 257, length 0

重传队列上的队首`sk_buff`被取下, 在途报文减少
```
tp->retrans_out = 0 
tp->sacked_out = 0  
tp->lost_out = 0 
tp->packets_out = 4   // [1001:2001] [2001:3001] [3001:4001] [4001:5001]
```

##### 2nd ack 到达

>  192.0.2.1.58393 > 192.168.165.149.9996: Flags [.], ack 1001, win 257, options [sack 1 {2001:3001},nop,nop], length 0

2nd ack 携带了一个 SACK block. 因此, [2001:3001]对应的`sk_buff`将 SACKED 标记

```
tp->retrans_out = 0 
tp->sacked_out = 1   // [2001:3001]
tp->lost_out = 0 
tp->packets_out = 4  // [1001:2001] [2001:3001] [3001:4001] [4001:5001]
```

##### 3rd ack 到达

> 192.0.2.1.58393 > 192.168.165.149.9996: Flags [.], ack 1001, win 257, options [sack 1 {4001:5001},nop,nop], length 0

2nd ack 携带了一个 SACK block. 因此, [4001:5001]对应的`sk_buff` 被标记上 SACKED

```
tp->retrans_out = 0 
tp->sacked_out = 2   // [2001:3001] [4001:5001]
tp->lost_out = 0 
tp->packets_out = 4  // [1001:2001] [2001:3001] [3001:4001] [4001:5001]
```
##### 重传

发送端进行重传

> 192.168.165.149.9996 > 192.0.2.1.58393: Flags [P.], seq 1001:2001, ack 1, win 502, length 1000

重传队列上, [1001:2001] 被标记上 RETRANS 和 LOST. [3001:4001] 被标记上 LOST

```
tp->retrans_out = 1  // [1001:2001]
tp->sacked_out = 2   // [2001:3001] [4001:5001]
tp->lost_out = 2     // [1001:2001] [3001:4001] 
tp->packets_out = 4  // [1001:2001] [2001:3001] [3001:4001] [4001:5001]
```

##### 4th ack 到达

> 192.0.2.1.58393 > 192.168.165.149.9996: Flags [.], ack 3001, win 257, length 0

4th ack 确认到 3001 之前的报文, 因此, 可以将 [1001:2001] 和 [2001:3001] 都从重传队列取下

```
tp->retrans_out 0    
tp->sacked_out 1,   //  [4001:5001]
tp->lost_out 1,     //  [3001:4001] 
tp->packets_out 2   //  [3001:4001] [4001:5001]
```

##### 重传

发送端进行重传

> 192.168.165.149.9996 > 192.0.2.1.58393: Flags [P.], seq 3001:4001, ack 1, win 502, length 1000

重传队列上, [3001:4001] 被标记上 RETRANS

```
tp->retrans_out 1,  //  [3001:4001]    
tp->sacked_out 1,   //  [4001:5001]
tp->lost_out 1,     //  [3001:4001] 
tp->packets_out 2   //  [3001:4001] [4001:5001]
```

##### 5th ack 到达

> 192.0.2.1.58393 > 192.168.165.149.9996: Flags [.], ack 5001, win 257, length 0

所有报文均得到确认. 重传队列上再无`sk_buff`

```
tp->retrans_out = 0 
tp->sacked_out = 0  
tp->lost_out = 0 
tp->packets_out = 0
```

#### 附录

实验使用的 packetdrill 脚本
```
0 `sysctl -q net.ipv4.tcp_sack=1`
0 `sysctl -q net.ipv4.tcp_recovery=0`
+0  socket(..., SOCK_STREAM, IPPROTO_TCP) = 3
+0 setsockopt(3, SOL_SOCKET, SO_REUSEADDR, [1], 4) = 0
+0 bind(3, ..., ...) = 0
+0 listen(3, 1) = 0

// 3-way handshake
+0 < S 0:0(0) win 32792 <mss 1000,sackOK,nop,nop,nop,wscale 7>
+0 >  S. 0:0(0) ack 1 win 64240 <mss 1460,nop,nop,sackOK,nop,wscale 7>
//0.100 > S. 0:0(0) ack 1 <mss 1460,nop,nop,sackOK,nop,wscale 6>
+.1 < . 1:1(0) ack 1 win 257
+0 accept(3, ..., ...) = 4

// Write extra 7 data segments.
+0 write(4, ..., 1000) = 1000
+0 > P. 1:1001(1000) ack 1
+0 write(4, ..., 1000) = 1000
+0 > P. 1001:2001(1000) ack 1
+0 write(4, ..., 1000) = 1000
+0 > P. 2001:3001(1000) ack 1
+0 write(4, ..., 1000) = 1000
+0 > P. 3001:4001(1000) ack 1
+0 write(4, ..., 1000) = 1000
+0 > P. 4001:5001(1000) ack 1

// 1st ack 
+0 < . 1:1(0) ack 1001 win 257

// 3rd ack
+0 < . 1:1(0) ack 1001 win 257 <sack 2001:3001,nop,nop>
// 4th ack
+0 < . 1:1(0) ack 1001 win 257 <sack 4001:5001,nop,nop>

+0.1 > P. 1001:2001(1000) ack 1
+0 < . 1:1(0) ack 3001 win 257

+0.1 > P. 3001:4001(1000) ack 1
+0 < . 1:1(0) ack 5001 win 257
```
