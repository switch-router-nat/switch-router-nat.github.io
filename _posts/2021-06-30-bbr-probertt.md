---
layout    : post
title     : "TCP BBR 实现原理--Probe RTT"
date      : 2021-06-30
lastupdate: 2021-06-30
categories: Network(Kernel)
---

<p align="center"><img src="/assets/img/public/cong.png"></p>

> 本系列使用内核代码版本均为 [4.9.1](https://elixir.bootlin.com/linux/v4.9.1/source/net/ipv4/tcp_rate.c) , 也就是 BBR 刚进入内核的版本
>
> 特别注意: 本文贴出代码是经过了简化, 以方便理解主要流程.

本文的中心是 Probe RTT.BBR 每 10s 中就有 200ms 处于的这个过程中.

#### 输入与输出

BBR 从 TCP 拿到 rate sample 信息, 其中就包含了 TCP 测量的 RTT 数据，BBR 处理后，通过设置拥塞窗口影响 TCP 的报文发送

<p align="center"><img src="/assets/img/bbr-probertt/pic1.png"></p>

#### RTT 的来源

BBR 始终保存当前连接最小的 rtt,  而 rtt 的原始数据由 TCP 在收到 (s)ack 后估算的 rs->rtt\_us 提供

注意,  BBR 使用是真实探测的值, 而不是经过平滑的 srtt. 主要原因是 srtt 只管 ack, 对 sack  视而不见，而 BBR 则对 (s)ack 一视同仁.

#### 刷新 RTT

每次 BBR 收到输入的 rtt , 都会与已保存的 min\_rtt 相比较,  如果输入值更小， 就保存新值,  并记录此刻的时间戳

```c
void bbr_update_min_rtt(struct sock *sk, const struct rate_sample *rs) {
    ......
    if (rs->rtt_us >= 0 &&
        (rs->rtt_us <= bbr->min_rtt_us)) {
        bbr->min_rtt_us = rs->rtt_us;
        bbr->min_rtt_stamp = tcp_time_stamp;
    }
    ......
}
```
#### 进入 Probe RTT 模式

一旦 10s 内, BBR 保存的最小 min\_rtt 没有刷新, BBR 就强制进入 Probe RTT 模式. 将 `bbr->probe_rtt_done_stamp`  表示 PROBE RTT 结束时间戳,  它之后会在 Probe RTT 真正启动时, 设置为启动时间 + 200ms ( PROBE Rtt 阶段至少持续 200ms)

```c
void bbr_update_min_rtt(struct sock *sk, const struct rate_sample *rs) {
    ...
    filter_expired = after(tcp_time_stamp, bbr->min_rtt_stamp + bbr_min_rtt_win_sec * HZ);
    if (filter_expired & !bbr->idle_restart && bbr->mode != BBR_PROBE_RTT) { 
        bbr->mode = BBR_PROBE_RTT;  /* dip, drain queue */
        ...
        bbr->probe_rtt_done_stamp = 0;
    }
    ...
}
```
#### 启动 Probe RTT 200ms 计时

注意, 进入 Probe RTT 模式后，内核会为其设置退出时机,  根据设计, BBR 的 Probe RTT 至少需要持续 200ms, 而这个 200ms 是从 in flight 的报文不大于 `bbr_cwnd_min_target`  (值为 4 ) 才开始计时。如果此时 in flight 的报文超过4, 则此时虽然进入了Probe RTT 模式,  但却不会算在 200ms 内。只有当这两个条件都满足时, 200ms 计时开始, `bbr->probe_rtt_done_stamp`被设置为 200ms 以后, 再将此时的 `tp->delivered` 拍照保存到 `bbr->next_rtt_delivered`,

```c
void bbr_update_min_rtt(struct sock *sk, const struct rate_sample *rs) {
    ...
    if (bbr->mode == BBR_PROBE_RTT) {
        /* Maintain min packets in flight for max(200 ms, 1 round). */
        if (!bbr->probe_rtt_done_stamp &&								// 启动200ms计时条件1
            tcp_packets_in_flight(tp) <= bbr_cwnd_min_target) {         // 启动200ms计时条件2 
            bbr->probe_rtt_done_stamp = tcp_time_stamp +
                msecs_to_jiffies(bbr_probe_rtt_mode_ms);
            bbr->probe_rtt_round_done = 0;
            bbr->next_rtt_delivered = tp->delivered;
        } 
        ...
    }
}
```
#### Probe RTT 生效

当 BBR 的进入 Probe RTT 模式, 会严格限制报文发送速率,  这是通过设置 TCP 的 `tp->snd_cwnd` (拥塞窗口) 实现的, 把它设置到不超过 `bbr_cwnd_min_target` 这样这条连接的报文会迅速减少, 当减少到 `bbr_cwnd_min_target` 后, 即可设置 200ms 的退出时机

```
bbr_main
 |-- bbr_set_cwnd
 |-- if (bbr->mode == BBR_PROBE_RTT)  /* drain queue, refresh min_rtt */
     tp->snd_cwnd = min(tp->snd_cwnd, bbr_cwnd_min_target);
```

#### Probe RTT 退出

BBR 退出 Probe RTT 模式需要满足两个条件: 1. 已经经过了一轮报文往返 (收到了一个在启动 Probe RTT 模式之后发送的报文的 ack)  2. 持续了 200ms

BBR 在开始 200ms 计时的时候, 将`tp->delivered` 拍照保存到 `bbr->next_rtt_delivered`. 因此如果之后确认一个启动计时之后的报文成功送达, 则可确认已经经过了一个完整报文往返周期。下面的代码中 `bbr->round_start = 1`, 表示这是新的报文
```c
bbr_update_bw(struct sock *sk, const struct rate_sample *rs) {
    bbr->round_start = 0;
    if (!before(rs->prior_delivered, bbr->next_rtt_delivered)) {
        bbr->next_rtt_delivered = tp->delivered;
        bbr->rtt_cnt++;
        bbr->round_start = 1;
        ...
    }
    ...
}
```
当 Probe RTT 进行退出检查时, 会同时检测两个条件是否满足, 如果满足, 就使用 `bbr_reset_mode` 返回进入 Probe RTT 模式之前的模式

```c
void bbr_update_min_rtt(struct sock *sk, const struct rate_sample *rs) {
	...
    if (bbr->mode == BBR_PROBE_RTT) {
	   if (...)
	   ...
       else if (bbr->probe_rtt_done_stamp) {
			if (bbr->round_start)
				bbr->probe_rtt_round_done = 1;
			if (bbr->probe_rtt_round_done &&
                after(tcp_time_stamp, bbr->probe_rtt_done_stamp)) {
                bbr->min_rtt_stamp = tcp_time_stamp;
                bbr->restore_cwnd = 1;  /* snap to prior_cwnd */
                bbr_reset_mode(sk);
            }
    }
}
```

### 总结

本文分析了内核 BBR 中 Probe RTT 模式的整个运行过程, 包括启动、生效、结束几个过程的条件和时机.

