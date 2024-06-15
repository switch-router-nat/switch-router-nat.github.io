---
layout    : post
title     : "tcp_tw_reuse 的原理和实现"
date      : 2021-06-30
lastupdate: 2021-06-30
categories: Network(Kernel)
---

<p align="center"><img src="/assets/img/public/tcp.png"></p>
    
### 历史与现状

这是一个从 2.4 版本就存在的内核选项. 

> tcp_tw_reuse - BOOLEAN
>   Allow to reuse TIME-WAIT sockets for new connections when it is
>   safe from protocol viewpoint. Default value is 0.

使能后, tcp 可以复用正处于 TIME-WAIT 状态的 socket 的四元组 (前提是 safe)

最初它只有 0 和 1 两个选择, 2018 年的一个 [commit](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=79e9fed460385a3d8ba0b5782e9e74405cb199b1) 将其取值范围扩展为 0，1，2.

```patch

diff --git a/Documentation/networking/ip-sysctl.txt b/Documentation/networking/ip-sysctl.txt
index 924bd51327b7a..6841c74eac007 100644
--- a/Documentation/networking/ip-sysctl.txt
+++ b/Documentation/networking/ip-sysctl.txt
@@ -667,11 +667,15 @@ tcp_tso_win_divisor - INTEGER
 	building larger TSO frames.
 	Default: 3
 
-tcp_tw_reuse - BOOLEAN
-	Allow to reuse TIME-WAIT sockets for new connections when it is
-	safe from protocol viewpoint. Default value is 0.
+tcp_tw_reuse - INTEGER
+	Enable reuse of TIME-WAIT sockets for new connections when it is
+	safe from protocol viewpoint.
+	0 - disable
+	1 - global enable
+	2 - enable for loopback traffic only
 	It should not be changed without advice/request of technical
 	experts.
+	Default: 2
```

### 绕不开的 TIME-WAIT

关于 `TIME-WAIT` 状态, 虽然不是本文的重点, 但它毕竟是 `tcp_tw_reuse` 存在的因, 绕不过去.

这里只列出 `TIME-WAIT` 状态存在的意义, [参考资料](https://vincent.bernat.ch/en/blog/2014-tcp-time-wait-state-linux):

> 1. 防止由于网络delay,相同四元组的旧连接报文污染新连接
> 2. 用于响应对端在 4-way 挥手过程中可能重传的 FIN-ACK 报文.

rfc793 中`TIME-WAIT`状态要持续 2*MSL (240s), 这么长时间足够在工程上认为所有该连接的报文都消失了.

而 Linux 内核一直从采用的是 60s. 

```
#define TCP_TIMEWAIT_LEN (60*HZ) /* how long to wait to destroy TIME-WAIT
                                  * state, about 60 seconds     */
```

无论`TIME-WAIT`状态持续的时间是多少, 对我们而言, 

`TIME-WAIT`状态的实际意义是: 在状态持续时间内, 该 socket 的四元组仍然处于<mark>正使用</mark>的状态

### tcp_tw_reuse 的意义

站在连接发起端的角度, 在选择本地端口时, 必须跳过处于`TIME-WAIT`状态的端口. 

因此，一旦连接发起端在短时间大量建立再关闭与目标服务器的连接, 就会出现`TIME-WAIT`状态的连接越来越多, 

这些连接要持续 60s 才会被释放, 这就导致可用的闲置端口越来越少, 最终后续的连接将建立失败.

而 `tcp_tw_reuse` 的意义就是在<mark>保证安全的情况下, 连接发起端可以复用处于TIME-WAIT状态的连接的四元组</mark>

### tcp_tw_reuse 的实现

> 本文以 5.10 版本内核代码为基础

TCP 连接发起端在选择本地源端口时, 将会在端口选择范围内依次尝试作为候选端口, 候选端口在 `__inet_check_established()` 检查是否处于正使用状态.

> 注意: TIME-WAIT 状态的 socket 和 ESTABLISHED 状态的 sock 都会放在 ehash_bucket 中, 区别在于前者放的是 inet_timewait_sock, 后者是完整的 inet_sock

```c
inet_hash_connect
 |- __inet_hash_connect
   |- 寻找可用的 port, 进行 __inet_check_established 检查
   
__inet_check_established (...) {
    ...
    if (likely(inet_match(net, sk2, acookie, ports, dif, sdif))) {
			if (sk2->sk_state == TCP_TIME_WAIT) {
				tw = inet_twsk(sk2);
				if (twsk_unique(sk, sk2, twp))
					break;
			}
			goto not_unique;
		}
    ...
}
```

对 TIME-WAIT 状态的 sock, 将会调用 `twsk_unique()` 进行额外检查. 该函数返回 1 表示可以使用, 0 表示不可使用.

```c
twsk_unique
 |-- tcp_twsk_unique
 
int tcp_twsk_unique(struct sock *sk, struct sock *sktw, void *twp) {

    int reuse = READ_ONCE(sock_net(sk)->ipv4.sysctl_tcp_tw_reuse);
    const struct inet_timewait_sock *tw = inet_twsk(sktw);
	const struct tcp_timewait_sock *tcptw = tcp_twsk(sktw);
	struct tcp_sock *tp = tcp_sk(sk);

    if (reuse == 2) {
        ...
        if (!loopback)
			reuse = 0;
    }
    
    if (tcptw->tw_ts_recent_stamp &&
	    (!twp || (reuse && time_after32(ktime_get_seconds(),
					    tcptw->tw_ts_recent_stamp)))) {
	    ...
		return 1;
	}
	return 0;
}
```

这里再根据 `tcp_tw_reuse` 的不同值进行不同的处理: 

- 0: 始终返回 0, 即不能复用该端口;
- 1: 旧连接启用了 timestamp 选项, 且当前时刻超过了旧连接的最后一个报文 1s (因为时间戳分辨率为1s), 就可以复用;
- 2: 只有向本机发起的连接才继续考虑复用.

启用`tcp_tw_reuse`在事实上会将原本持续 60s 的 `TIME-WAIT` 状态缩短为 1s. 

如此一来, TCP 短时间能建立的短连接数量将会大幅提升.

### tcp_tw_reuse 的风险

短连接数量提升是启动`tcp_tw_reuse`带来的好处, 而其带来的风险则可以从`TIME-WAIT`存在的意义来考虑. 

> 1. 防止由于网络delay,相同四元组的旧连接报文污染新连接.

由于`tcp_tw_reuse`的使用前提是旧连接使用了时间戳, 因此即使旧连接报文到达, 也能通过时间戳进行识别.

> 2. 用于响应对端在 4-way 挥手过程中可能重传的 FIN 报文.

<p align="center"><img src="/assets/img/tcp-tw-reuse/pic1.png"></p>

当本端响应对端 FIN 的 ACK 报文丢失时, 对端会重传 FIN 报文. 如果`tcp_tw_reuse`启动后重新复用该端口, 就会在`SYN-SENT`状态收到对端重传的 FIN 报文.

新连接的建立会经历 RST 报文和 SYN 报文的重传，最终新连接也能建立.

虽然这不是最正常的方式, 但我认为其实也能接受.

### 总结

1. `tcp_tw_reuse`适合配置在大量短连接主动关闭端
2. `tcp_tw_reuse`依赖启动时间戳, 这是为了弥补取消 2MSL 失去的安全性
