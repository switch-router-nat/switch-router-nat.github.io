---
layout    : post
title     : "TCP 窗口最大值为什么是1GiB"
date      : 2023-03-15
lastupdate: 2023-03-15
categories: Network(others)
---

<p align="center"><img src="/assets/img/public/tcp.png"></p>

这个看上去理所当然的问题可能并没有那么简单...

```
        0                   1                   2                   3   
        0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 
       +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
       |          Source Port          |       Destination Port        |
       +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
       |                        Sequence Number                        |
       +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
       |                    Acknowledgment Number                      |
       +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
       |  Data |           |U|A|P|R|S|F|                               |
       | Offset| Reserved  |R|C|S|S|Y|I|            Window             |
       |       |           |G|K|H|T|N|N|                               |
       +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
       |           Checksum            |         Urgent Pointer        |
       +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
       |                    Options                    |    Padding    |
       +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
       |                             data                              |
       +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
       
                                TCP Header Format
```

如果从 TCP Header 的`Window`字段看, 它占 16 bits, 表示接收通告窗口的最大值为 2^16 = 64KB.

考虑到 Window Scale Option 的存在, 其定义 shift.cnt 最大为 14, 因此接收窗口的最大值可达 2^(14+16) = 1 GiB

```
    TCP Window Scale option (WSopt):

       Kind: 3

       Length: 3 bytes

              +---------+---------+---------+
              | Kind=3  |Length=3 |shift.cnt|
              +---------+---------+---------+
                   1         1         1

```

这就是 1 GiB 的来源, 但这个数值是建立在 shift.cnt 最大值只能为 14 的前提下. 问题是这个最大值为什么不能是 15、16 或者更大呢? 这样窗口大小也会扩大到 2 GiB、4 GiB...

我们知道 TCP 的序号是 32 bits. 它能表示的范围是 0 ~ 4GiB. 

#### 如果窗口大小为 4 GiB 会有什么问题?

接收端的视角下, 收到的报文的序号永远都会处于窗口内, 接收端将因此无法分辨 old 报文和 new 报文.

举个例子. 某一时刻, 接收端窗口左边沿 RCV.NXT = a, 如果此时收到了一个起始序号 SEG.Seq = b 的报文

<p align="center"><img src="/assets/img/tcp-max-window/pic1.png"></p>

它将无法区分这个 SEG 是需要接收的数据 (领先窗口左边沿 4G - a + b) 还是一个过气的报文 (落后窗口左边沿 a - b) ,因此, 窗口大小不能为 4 GiB.

#### 如果窗口大小为 2 GiB 会有什么问题?

我们先回顾下 TCP 的发送窗口与接收窗口

一般说来, 正常的 TCP 传输过程中, 发送窗口总是在\*\*<mark>追赶</mark>\*\*接收窗口.

举个例子:

阶段1: 发送方和接收方窗口大小为6, 且窗口处于对齐状态，发送方发送的3个报文(15\~17)在途

<p align="center"><img src="/assets/img/tcp-max-window/pic2.png"></p>

阶段2: 接收方收到15\~17报文, 推进接收窗口, 并发送应答报文，此时发送窗口和接收窗口出现了错位

<p align="center"><img src="/assets/img/tcp-max-window/pic3.png"></p>

阶段3: 发送方收到应答报文，推进发送窗口, 窗口再次对齐

<p align="center"><img src="/assets/img/tcp-max-window/pic4.png"></p>

也就是说，发送端窗口和接收端窗口始终进行着: 对齐->错位->对齐->错位->

最大能错位多少呢 ？ 答案是一整个窗口. 这个来自于 [rfc7323](https://www.rfc-editor.org/rfc/rfc7323.txt)

> since the sender and receiver windows can be out of phase by at most the window size

考虑极端情况, 发送端窗口和接收端窗口大小都为 2 GiB, 在某一时刻, 它们刚好完全错位, 两个窗口瓜分整个 4 GiB.

<p align="center"><img src="/assets/img/tcp-max-window/pic4.png"></p>

发送端迟迟收不到应答，重传序号为 \[0,1000] 的报文:

<p align="center"><img src="/assets/img/tcp-max-window/pic5.png"></p>

发送端收到应答，推进发送窗口与接收端窗口同步:

<p align="center"><img src="/assets/img/tcp-max-window/pic6.png"></p>

发送端发送新的数据, 序号为\[2G,2G+1000]:

<p align="center"><img src="/assets/img/tcp-max-window/pic7.png"></p>

网络发生乱序, 报文\[2G\~2G+1000] 比重传报文 \[0,1000] 更快到达接收端，接收端收到后推进窗口到 \[2G+1000, 1000]:

<p align="center"><img src="/assets/img/tcp-max-window/pic8.png"></p>

重传报文 \[0,1000]到达, 此时它处于接收窗口内, 因此会被误当做有效报文.

那么窗口大小在 1 GiB 到 2 GiB 可以吗 ? 比如 1.25 GiB ?

rfc7323 是这么说的:

> To insure that new data is never mistakenly considered old and vice versa, the left edge of the sender's window has to be at most 2^31 away from the right edge of the receiver's window.  The same is true of the sender's right edge and receiver's left edge.  Since the right and left edges of either the sender's or receiver's window differ by the window size, and since the sender and receiver windows can be out of phase by at most the window size, the above constraints imply that two times the maximum window size must be less than 2^31, or max window < 2^30

其中提到发送窗口的左边沿与接收窗口的右边沿不能超过 2^31 (2 GiB), 又由于它们可以完全错开一个完整的窗口，这样算下来, 窗口的最大值就是 2^31 / 2 = 2^30 (1 GiB)

虽然 rfc 这样说, 但我现在还是不清楚为什么 1.25 GiB 不可以, 毕竟 2 GiB 时的问题并不会存在....难道仅仅是因为 shift.cnt 需要为整数，由于15不行, 因此只能14?

以后如果想起再考虑吧....