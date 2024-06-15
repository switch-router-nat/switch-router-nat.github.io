---
layout    : post
title     : "内核TCP拥塞控制框架的两次演进"
date      : 2021-03-02
lastupdate: 2021-03-02
categories: Network(Kernel)
---

<p align="center"><img src="/assets/img/public/cong2.png"></p>

今天的主题是内核 TCP 拥塞控制实现的两次关键变化. 

#### 第一次 拥塞控制算法插件化 2005.1.24

[Add pluggable congestion control algorithm infrastructure.](https://github.com/torvalds/linux/commit/317a76f9a44b437d6301718f4e5d08bd93f98da7)

在这次 commit 之前, 内核已经支持了 reno、vegas、westwood、bic 拥塞控制算法. 但它们是直接嵌入在 tcp 的主路径中. 
```c
#define tcp_is_vegas(__tp)	((__tp)->adv_cong == TCP_VEGAS)
#define tcp_is_westwood(__tp)	((__tp)->adv_cong == TCP_WESTWOOD)
#define tcp_is_bic(__tp)	((__tp)->adv_cong == TCP_BIC)
....

static inline __u32 tcp_recalc_ssthresh(struct tcp_sock *tp)
{
	if (tcp_is_bic(tp)) {
	 ...
}

static inline void tcp_cong_avoid(struct tcp_sock *tp, u32 ack, u32 seq_rtt)
{
	if (tcp_vegas_enabled(tp))
		vegas_cong_avoid(tp, ack, seq_rtt);
	else
		reno_cong_avoid(tp);
}
```
这种直接调用而非通过 "注册-回调" 的形式. 会使得 tcp 的核心文件去引用各个拥塞控制算法, 并且还使得 tcp 主路径上出现大量的判断分支, 且它们还会随着支持的拥塞控制算法增多而增多.

而这次 commit 的目的就是将拥塞控制算法插件化, 解除它们与 tcp 核心代码之间的耦合
> Allow TCP to have multiple pluggable congestion control algorithms.
Algorithms are defined by a set of operations and can be built in
or modules. 

最重要的, 定义了 `struct tcp_congestion_ops`, 各个拥塞控制算法实现自己的结构, 并注册到内核中.

这样, tcp 核心主路径就只需要触发回调即可实现拥塞控制.
```c
static inline void tcp_cong_avoid(struct tcp_sock *tp, u32 ack, u32 rtt,
				  u32 in_flight, int good)
{
	tp->ca_ops->cong_avoid(tp, ack, rtt, in_flight, good);
	tp->snd_cwnd_stamp = tcp_time_stamp;
}
```

#### 第二次 cong_control完全接管  2016.9.21

[tcp: new CC hook to set sending rate with rate_sample in any CA state](https://github.com/torvalds/linux/commit/c0402760f565ae066621ebf8720a32fba074d538)

这个 commit 为 `struct tcp_congestion_ops` 添加了一个回调函数方法
```diff
struct tcp_congestion_ops {
	u32 (*tso_segs_goal)(struct sock *sk);
	/* returns the multiplier used in tcp_sndbuf_expand (optional) */
	u32 (*sndbuf_expand)(struct sock *sk);
+	/* call when packets are delivered to update cwnd and pacing rate,
+	 * after all the ca_state processing. (optional)
+	 */
+	void (*cong_control)(struct sock *sk, const struct rate_sample *rs);
	/* get info for inet_diag (optional) */
	size_t (*get_info)(struct sock *sk, u32 ext, int *attr,
			   union tcp_cc_info *info);
```

这个回调函数在`tcp_cong_control`中接管 tcp 的拥塞控制. 新的拥塞控制算法可以实现这个函数, 以达到对拥塞控制更大的掌控力. 在这之前, pacing rate 都是由 tcp 统一计算, 而现在, 可以由拥塞控制算法自个儿计算了. BBR 就是这么做的.

```diff
static void tcp_cong_control(struct sock *sk, u32 ack, u32 acked_sacked,
			     int flag)
			     int flag, const struct rate_sample *rs)
{
+	const struct inet_connection_sock *icsk = inet_csk(sk);
+
+	if (icsk->icsk_ca_ops->cong_control) {
+		icsk->icsk_ca_ops->cong_control(sk, rs);
+		return;
+	}

	if (tcp_in_cwnd_reduction(sk)) {
		/* Reduce cwnd if state mandates */
		tcp_cwnd_reduction(sk, acked_sacked, flag);
	} else if (tcp_may_raise_cwnd(sk, flag)) {
		/* Advance cwnd if state allows */
		tcp_cong_avoid(sk, ack, acked_sacked);
	}
	tcp_update_pacing_rate(sk);
}
```

#### 更多...

更有意思是. 经过这两个 commit, 我们完全可以编写自己的 tcp 拥塞控制算法, 编译成 ko, 然后注册进内核. 大概是下面这个模样

```c
...

static struct tcp_congestion_ops tcp_mycong_ops = {
    .flags = TCP_CONG_NON_RESTRICTED,
    .name = "mycong",
    .owner = THIS_MODULE,
    .init = mycong_init,
    .cong_control = mycong_main,
    .undo_cwnd = mycong_undo_cwnd,
    .ssthresh = mycong_ssthresh,
};

static int __init mycong_register(void)
{
    BUILD_BUG_ON(sizeof(struct mycong) > ICSK_CA_PRIV_SIZE);
    ...
    return tcp_register_congestion_control(&tcp_mycong_ops);
}

static void __exit mycong_unregister(void)
{
    tcp_unregister_congestion_control(&tcp_mycong_ops);
}

module_init(mycong_register);
module_exit(mycong_unregister);
```