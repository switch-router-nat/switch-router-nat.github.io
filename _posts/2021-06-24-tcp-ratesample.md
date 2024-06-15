---
layout    : post
title     : "理解 TCP rate sample"
date      : 2021-06-24
lastupdate: 2021-06-24
categories: Network(Kernel)
---

<p align="center"><img src="/assets/img/tcp-ratesample/fm.png"></p>

内核中的 TCP rate sample 被 bbr 这样的拥塞控制算法使用，它总共也就不到200行代码，其中一半还是注释，但理解起来可能还是需要花点力气。

本文将尝试解释它的实现原理.

> 本文使用的内核代码版本是 [4.9.1](https://elixir.bootlin.com/linux/v4.9.1/source/net/ipv4/tcp_rate.c)

一句话概括, rate sample 的结果是**一条流**在\*\* interval \*\*时间内发送的报文被网络成功 **delivered** (成功送达)的数目。

比如, 5s 内, 成功送达了 10个报文、10s 内成功送达了 15 个报文 .... 等等.

内核的相关实现一共就3个函数, 对应 "起点拍照" , "快照填充rate sample" , "计算rate sample" 三个阶段

#### 起点拍照  (tcp\_rate\_skb\_sent)

发送 tcp 报文时，会将**当前时间**和当前**已 delivered 报文**的数目记到 skbuff 的 CB 中，这个过程俗称 "拍快照" (Snatshot)。在报文没有被 ack 之前，该 skbuff 会一直挂在重传队列上; 而当收到 ack 应答时，将当前时刻的信息(如已delivered的报文数目) 与 skbuff 中的数据进行比较，就得到了 时间+数目 , 即 rate sample 的结果。

快照拍摄的对象是 tcp 连接的 tcp sock 上下面这些字段中的部分

```diff
struct tcp_sock {
+	u32	delivered;	/* Total data packets delivered incl. rexmits */
	u32	lost;		/* Total data packets lost incl. rexmits */
+	u32	app_limited;	/* limited until "delivered" reaches this val */
+	struct skb_mstamp first_tx_mstamp;  /* start of window send phase  */
+	struct skb_mstamp delivered_mstamp; /* time we reached "delivered" */
	u32	rate_delivered;    /* saved rate sample: packets delivered */
	u32	rate_interval_us;  /* saved rate sample: time elapsed */
	...
}
```

其中 重要的是下面三个

- delivered 表示截止此刻该条连接已经成功送达的报文数目
- delivered\_mstamp :达到送达 delivered 个报文的时间戳
- first\_tx\_mstamp  采样周期开始的时刻, 也就是 interval 的起始时间戳

前两个的意思很明确，不多说。但 first\_tx\_mstamp 比较有意思. 它遵守以下规则:

1.发送报文时, 如果网络中没有这条流没有 ack 的数据包(说明这条流处于闲置状态)，就以当前时刻作为起点, first\_tx\_mstamp 顾名思义意思为 "发送第一个报文时的时间戳"
2.收到 ack 后, 将 skb 从重传队列上移除, 将这个 skb 当初发送时刻的时间戳作为起点.

画成时间序列图如下

<p align="center"><img src="/assets/img/tcp-ratesample/pic1.png"></p>

图中以时间轴为单位，第一行表示发送报文i, 第二行表示收到报文i 对应的 ack,  后面几行表示对应时刻 tcp\_sock 的上述三个字段的值.

#### 快照填充rate sample (tcp\_rate\_skb\_delivered)

这一步是在收到报文的ack时被调用, 内核将根据报文中在当初发送时保存的快照, 填充 rate sample 的起点信息

<p align="center"><img src="/assets/img/tcp-ratesample/pic2.png"></p>

注意, 其中的 interval 计算公式为 发送时刻 - 采样周期开始的时刻,  这个 interval 也被成为 send phase 的 interval

#### 计算 rate sample  (tcp\_rate\_gen)

这个过程的有最重要的输入参数是当前时刻的已送达的报文数目 `tp->delivered`.

<p align="center"><img src="/assets/img/tcp-ratesample/pic3.png"></p>

如上图所示, 每当收到 ack, 协议栈就会计算出一个 rate sample. 其中`rs->delivered`没有异议, 而`rs->interval`取的是 snd\_us 和 ack\_us 的较大值. 前者是的意义是通过 send pipeline 估计速率，后者是通过 ack pipeline 估计.

关于这一点, 在文件开头的注释中有解释

        /* The bandwidth estimator estimates the rate at which the network
         * can currently deliver outbound data packets for this flow. At a high
         * level, it operates by taking a delivery rate sample for each ACK.
         *
         * A rate sample records the rate at which the network delivered packets
         * for this flow, calculated over the time interval between the transmission
         * of a data packet and the acknowledgment of that packet.
        */

意思是 bandwidth estimator 会同时使用两种 interval 进行计算: 1.使用发送时的时刻计算 2.通过ack到达的时刻计算

         * Specifically, over the interval between each transmit and corresponding ACK,
         * the estimator generates a delivery rate sample. Typically it uses the rate
         * at which packets were acknowledged. However, the approach of using only the
         * acknowledgment rate faces a challenge under the prevalent ACK decimation or
         * compression: packets can temporarily appear to be delivered much quicker
         * than the bottleneck rate. Since it is physically impossible to do that in a
         * sustained fashion, when the estimator notices that the ACK rate is faster
         * than the transmit rate, it uses the latter:
         *    send_rate = #pkts_delivered/(last_snd_time - first_snd_time)
         *    ack_rate  = #pkts_delivered/(last_ack_time - first_ack_time)
         *    bw = min(send_rate, ack_rate)

一般说来, 我们使用 ack到达的时刻计算. 但有一种特别的情况，就是当出现 ack compression 时(一个 ack 应答多个 skb), 这个时候用 ack\_us 当做 interval 就可能出现计算出的带宽超过 bottleneck rate (物理链路的极限). 因此, 这种情况下,  内核使用 snd\_us 进行计算, 对应上面最后一张图的 ack4。
