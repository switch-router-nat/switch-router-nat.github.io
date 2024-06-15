---
layout    : post
title     : "tcp-brutal: 似控非控的拥塞控制算法"
date      : 2023-09-08
lastupdate: 2023-09-08
categories: Network(Kernel)
---

<p align="center"><img src="/assets/img/public/cong2.png"></p>

TCP 拥塞控制算法的本质是限制**<mark>自身</mark>**的发送速率，以缓解整个网络的拥堵. 如果参与网络的终端都践行这套规则. 那么谁也不会吃亏. 

但问题是，一个参与者或许可以通过破坏规则而获利(抢占其他参与者的带宽), 而如果每个参与者都不遵守规则，后果就是所有人都占不到便宜. 

目前还可以庆幸的是，尽管互联网上已经有形形色色的规则破坏者, 但它们的流量很小, 尚且动摇不了整个公平的根基...

本文的主题是 tcp-brutal 拥塞控制算法, 项目地址在[这里](https://github.com/apernet/tcp-brutal/tree/master) 本质上来说, 它也是规则破坏者的一员

一句话描述 tcp-brutal 拥塞控制算法: **<mark>"带宽充裕, 不会多要. 拥塞发生, 保证自身"</mark>**.

#### 实现原理

tcp-brutal 以内核模块的方式实现 (得益于[拥塞控制框架的两次演进](https://switch-router.gitee.io/blog/tcp-cong-2commit/)).

```
static struct tcp_congestion_ops tcp_brutal_ops = {
    .flags = TCP_CONG_NON_RESTRICTED,
    .name = "brutal",
    .owner = THIS_MODULE,
    .init = brutal_init,
    .cong_control = brutal_main,
    .undo_cwnd = brutal_undo_cwnd,
    .ssthresh = brutal_ssthresh,
};
```
最重要的是 `brutal_init` 和 `brutal_main`

`brutal_init` 主要职责是重写了 tcp 的 `setsockopt`: 新加上了一个私有的 TCP 选项 `TCP_BRUTAL_PARAMS`. 用户态程序可以使用这个选项设置 tcp-brutal 算法的参数.

它需要用户设置的参数只有预设速率 `rate` 和拥塞窗口增益 `cwnd_gain`, **<mark>并且这两个参数在运行过程中不会变化</mark>** (除非用户主动设置)

```c
struct brutal_pkt_info
{
    u64 sec;
    u32 acked;
    u32 losses;
};

struct brutal
{
    u64 rate;
    u32 cwnd_gain;

    struct brutal_pkt_info slots[PKT_INFO_SLOTS];
};
```

这里 tcp-brutal 还准备了3~5个 `slots` 记录丢包数量和应答数量, 用于计算丢包率, 1 个slot 可以统计 1s 内的报文数量, slot 重复循环使用.

`brutal_main` 就是计算发包成功率, 并调整 pacing 速率. 假设丢包率为 a, 预设的发包速率为 rate.

则最终计算的 pacing 速率 = rate / (1-a)

举个例子, 加速预设速率为 50Mbps, 丢包率为 20%, 则 pacing rate = 50 / (1-20%) = 62.5 Mbps 

也就是说, 在 tcp-brutal 眼中，不管网络信道拥塞成什么样子，我都始终都要保持预设的有效发送速率!

tcp-brutal 进行拥塞控制了吗? 似乎控了, 毕竟我设置了 pacing 的上边界; 又似乎没有, 面对拥塞, tcp-brutal 选择加大力度.

即使 tcp-brutal 要求用户设置速率, 并最多不超过可使用的带宽. 其逻辑是，既然我被承诺有这么大的带宽，那么这些带宽就是我应得的.

> 与 Hysteria 一样，Brutal 需要用户知道自己所处网络环境的带宽上限是多少

由此可见，设计一个'激进'的拥塞控制算法是多么容易! 多加窗，少减窗就可以了,  最'激进'的控制算法就是不要控制, 暴力发包就行. 

这不禁又让我想起 [kcp](https://github.com/skywind3000/kcp) 

> KCP是一个快速可靠协议，能以比 TCP 浪费 10%-20% 的带宽的代价，换取平均延迟降低 30%-40%，且最大延迟降低三倍的传输效果

哎, 差不多也是一个意思, 牺牲公平性，满足自己











