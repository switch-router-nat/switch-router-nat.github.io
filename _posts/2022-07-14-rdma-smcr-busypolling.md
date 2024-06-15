---
layout    : post
title     : "busypoll 模式能让 SMC-R 更快吗"
date      : 2022-07-14
lastupdate: 2022-07-14
categories: Network(Kernel)
---

在(SMC-R 加速 TCP)[https://switch-router.gitee.io/blog/rdma-smcr-acc/] 中我们提到了SMC-R能让使用普通 posix API 的网络应用在不加任何修改的情况下，也能享受 RDMA 带来的传输体验提升.

但通过与使用 verbs API 的程序实测比较以及分析, 我发现 SMC-R 并不能达到 verbs API 能达到的传输极限.

一个原因是: 当前 SMC-R 的实现无法支持 busy polling 模式.

### 四种传输模式的时延比较

这里我分别使用 qperf (模拟简单应用) 和 brpc (模拟一般应用) 进行ping-pong测试, 比较了 tcp 、smc-r、verbs 、 verbs-with-polling 四种方案的时延测试结果

结果如下所示(单位为微秒, 数值越小表示加速效果越好):

其中 场景1--qperf (size 10), 场景2--qperf (size 100), 场景3--brpc(size 0)  场景4--brpc

<p align="center"><img src="/assets/img/rdma-smcr-busypolling/pic1.png"></p>

从加速效果来看: verbs-with-polling >> verbs >= smc-r > tcp

可见, smc-r 虽然要比 verbs 性能稍微差一点, 但差距并不明显. 这一点也比较合理, 因为 smc-r 的实现原理就是将用户的 posix API 劫持, 然后在内核使用 verbs kernel API.相对于直接使用用户态 verbs API，路径的确长了一点.

至少我个人认为, smc-r 的这一点性能代价换来对应用的无侵入的优势 (使用verbs API需要改造应用)，还是很划算的。

但 verbs-with-polling 一骑绝尘的测试效果却有十分令人心动。polling 相较于 non-polling 在时延上的降低是可以理解的，因为付出了更多的 cpu 的代价.

那么, SMC-R 也可能支持 polling 模式吗?

### SMC-R 的 polling 模式探索

我想到了两种思路: 1.内核线程  2. 嵌入内核polling框架

#### 内核线程

SMC-R 内核模块创建一个内核线程专门用于 poll 建立的 cq, 而不必依赖原本在软中断中读取 cq 的元素.

但是很遗憾的是实测下来效果并不好: cpu 占用率符合预期上去了, 但 latency 却没有明显降低...

我想原因应该是, 虽然内核层面是变成了 polling, 但对于使用 epoll 的用户态应用, 实际还是异步通知模式.

#### 嵌入内核 polling 框架

稍微看了下 busy polling 的代码, 我发现这条路还是很难走通.

对于只使用 recv \ recvmsg 而不使用 epoll 的应用, 或许有机会. 我可以模仿 tcp 这个函数, 实现一套 smc sock 的 busy poll.

```c
int tcp_recvmsg(struct sock *sk, struct msghdr *msg, size_t len, int flags,
		int *addr_len)
{
	....
	if (sk_can_busy_loop(sk) &&
	    skb_queue_empty_lockless(&sk->sk_receive_queue) &&
	    sk->sk_state == TCP_ESTABLISHED)
+		sk_busy_loop(sk, flags & MSG_DONTWAIT);
```

但问题是, 绝大部分网络应用是直接或者间接使用 epoll 的. 而 epoll 的 busy poll loop 却是很难插手的. 我没有办法插入一段 smc-r 相关的逻辑..

```c
static bool ep_busy_loop(struct eventpoll *ep, int nonblock)
{
	unsigned int napi_id = READ_ONCE(ep->napi_id);

	if ((napi_id >= MIN_NAPI_ID) && net_busy_loop_on()) {
		napi_busy_loop(napi_id, nonblock ? NULL : ep_busy_loop_end, ep, false,
			       BUSY_POLL_BUDGET);
		if (ep_events_available(ep))
			return true;
		/*
		 * Busy poll timed out.  Drop NAPI ID for now, we can add
		 * it back in when we have moved a socket with a valid NAPI
		 * ID onto the ready list.
		 */
		ep->napi_id = 0;
		return false;
	}
	return false;
}
```

### 结束了吗

从结果来看, 这次探索是失败的, 但是我还有一个想法，如果 SMC-R 是在用户态是不是一切就是可能的呢, 当然，这时它就不是原本的 SMC-R 了.
