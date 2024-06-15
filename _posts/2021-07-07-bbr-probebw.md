---
layout    : post
title     : "TCP BBR 实现原理--Probe BW"
date      : 2021-07-07
lastupdate: 2021-07-07
categories: Network(Kernel)
---

<p align="center"><img src="/assets/img/public/cong.png"></p>

> 本系列使用内核代码版本均为 [4.9.1](https://elixir.bootlin.com/linux/v4.9.1/source/net/ipv4/tcp_rate.c) , 也就是 BBR 刚进入内核的版本
>
> 特别注意: 本文贴出代码是经过了简化, 以方便理解主要流程.

本文的中心是 Probe BW. BBR 在绝大部分时间都是处于这个状态. 这个状态下, BBR 会不断探测带宽是否变大. 如果变大, BBR 就可以设置更高的 pacing rate, 加速发包

### bw 的输入和输出

与 Probe Rtt 时一致, 计算 bw 的数据来源是 TCP 提供的 rate sample,  它提供了过去 xx 时间段内, 这条连接一共有 xx 个报文确认到达了对端。

经过数据处理, 最终 BBR 输出 pacing rate 和 cwnd 影响 TCP 数据发送

<p align="center"><img src="/assets/img/bbr-probebw/pic1.png"></p>

#### 计算实时 bw, 保存最大 bw

BBR 算法每次运行时, 都会计算最新的 BW. 同时, BBR 还会始终保存**过去一段时间内**最大的 BW .. 这里使用的 minmax 统计方法, 可见  [win-minmax算法](https://switch-router.gitee.io/blog/win-minmax/);
```c
bbr_main(struct sock *sk, const struct rate_sample *rs)
 |
bbr_update_model(struct sock *sk, const struct rate_sample *rs)
 |
bbr_update_bw(struct sock *sk, const struct rate_sample *rs) {
    bw = (u64)rs->delivered * BW_UNIT;
    do_div(bw, rs->interval_us);
    if (bw >= bbr_max_bw(sk)) {
        minmax_running_max(&bbr->bw, bbr_bw_rtts, bbr->rtt_cnt, bw);
    }
 }
```

bw 本身的计算很简单, 就是`delivered` 与 `interval_us` 的比值. 注意, 这里使用了 `BW_UNIT` 进行缩放。原因是, 如果不缩放, 则得到的结果数值精度太差了。

举个例子, 假设 rate sample 提供的信息是`delivered`为 1个报文,  `interval_us`值为1μs

按1个报文 1500 bytes 计算 bw

> bw = 1500 bytes / 1 μs = 12 Gbps;

这还是最小的结果, 因为要保证比值有意义, `delivered`的数值需要大于 `interval_us`.

因此 BBR 计算是会先将`delivered`放大`BW_UNIT` (放大`1<<24`倍). 这样一来, 最后算下来的比值的单位就是

> 12 Gbps / (1>>24) ~= 715 bps.

举个例子, 假设 rate sample 提供的信息是`delivered`为 10 个报文,  `interval_us`值为1000μs

> bw = 10 * 1<<24 / 1000 = 167772  

由于它的单位是 715bps, 它实际表示的带宽是

> 167772*715bps ~= 120Mbps 

计算出 bw 后, BBR 会将其按照 minmax 算法进行合并.

#### 使用 bw 计算 pacing rate

BBR 将 bw 换算成 pacing rate 控制报文发送速率, 后者的单位是 bytes/s

```c
static void bbr_set_pacing_rate(struct sock *sk, u32 bw, int gain)
{
    struct bbr *bbr = inet_csk_ca(sk);
    u64 rate = bw;

    rate = bbr_rate_bytes_per_sec(sk, rate, gain);
    ......
    sk->sk_pacing_rate = rate;
}

static u64 bbr_rate_bytes_per_sec(struct sock *sk, u64 rate, int gain)
{
    rate *= tcp_mss_to_mtu(sk, tcp_sk(sk)->mss_cache);
    ...
    rate *= USEC_PER_SEC;
    return rate >> BW_SCALE;
}
```
上面是换算的代码 (忽略其中的增益系数 gain) , 整体逻辑很清楚
```
bw = delivered / interval_us * (1<<24)

rate = bw * mtu * USEC_PER_SEC / (1<<24) = delivered * mtu / interval_second
```


