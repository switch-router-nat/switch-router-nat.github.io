---
layout    : post
title     : "TCP timestamp 选项那点事"
date      : 2021-04-05
lastupdate: 2021-04-05
categories: Network(Kernel)
---

<p align="center"><img src="/assets/img/public/tcp.png"></p>

TCP 最早在 [RFC1323](https://www.rfc-editor.org/rfc/rfc1323.html) [] 中引入了 timestamp 选项, 并在后来的 [RFC7323](https://www.rfc-editor.org/rfc/rfc7323.html) 中进行了更新。引入 timestamp 最初有两个目的：**1.更精确地估算报文往返时间(round-trip-time, RTT)** **2. 防止陈旧的报文干扰正常的连接**.

本文将以 RFC7323 为基础，介绍 timestamp 选项的应用场景和当前业界对其的一些讨论。

### 介绍

Timestamp 是作为一个 TCP 选项存在于 TCP 首部。如下图所示，一个 timestamp 选项需要占据
首部中的 10 个字节。
 
```
 Kind: 8
 Length: 10 bytes
 
          +-------+-------+---------------------+---------------------+
          |Kind=8 |  10   |   TS Value (TSval)  |TS Echo Reply (TSecr)|
          +-------+-------+---------------------+---------------------+
             1       1              4                     4
```

选项的核心数据是两个 32-bit 的时间戳字段.**TSval** 表示发送端发出该报文时的本地时间戳，
而 **TSecr** 则负责回放(Echo) 最近一次收到的对端报文中的 **TSval** 的值。
 
**TSval 和 TSecr 在数值上并没有绝对的大小关系**。TSval 是以本地的时钟为基准的，
而 TSecr 则是以对端的时钟为基准的。

以下是一个组典型的时间戳交互过程
```
             TCP  A                                     TCP B

                             <A,TSval=1,TSecr=120> ----->

                  <---- <ACK(A),TSval=127,TSecr=1>

                             <B,TSval=5,TSecr=127> ----->

                  <---- <ACK(B),TSval=131,TSecr=5>
```

启用 Timestamp 选项需要经过双方的协商，协商在三次握手时完成，如果协商成功,则在后续的报文中，
除了 RST 之外的所有报文均**必须**包含 Timestamp 选项。

### 虚拟时钟

前面提到过，TSval 的值是以本地时钟为基准的，更准确地说是来自一个虚拟的时间戳时钟，这个时钟的流逝必须
与真实时间成比例，但不需要一定与真实时间相同，同时它既不能走得太慢，也不能走得太快

不能走得太慢是为了能更准确地测量报文的 RTT。假设这个时钟 10s 才 tick 一下，那么对于 
往返时间为 1s 的 TCP 连接，一端发送报文之后，很有可能会发现收到对端的 ACK 报文中的 TSecr 和当前时钟
的值是一样的，这说明 RTT 为 0 ! 显然，这是十分荒谬的。

不能走的太快是为了防止时间戳回绕的干扰。TCP 协议规定的最大 MSL 为 255s, 显然一次时钟的循环必须大于这个值，
换算下来，32-bit 的时钟 tick 必须大于 59ns。否则，就无法区分两个时间戳之差是否经历了时间戳的回绕。

RFC 规定虚拟时钟的频率为每 1ms 到 1s 一个 tick。 按照 1ms 计算，32-bit 的时间戳回绕一次需要 24.8 天。

### TSecr 的选择

通常情况下，为报文填入 TSecr 几乎是无脑的，因为只需要填入对端上一个报文的 TSval 就行了，但有几种特殊场景需要注意

#### 1.Delay ACK

Delay ACK 是为了减少网络中的 pure ACK, 设计了一种接收端延时应答机制，当接收端收到一个数据报文时，不立即进行应答，
而是等待一段时间(通常为 40ms), 如果在这段时间内本端有数据也正好需要发送，则可以减少一个 pure ACK。
如果这段时间内又收到了后续其他报文，则接收端会累积起来，时间结束后一齐应答，这样也可以减少 pure ACK。

那么如果启用了 Delay ACK, 并且接收端收到了多个报文，这些报文的 TSval 不同，那么应该 Echo 哪一个报文的 TSval 呢? 

答案是：需要 Echo 最早收到的那个报文的 TSval , 因为只有这样，发送端测量的 RTT 才更加准确.

#### 2.报文丢失造成了接收端序号空洞

发送端发送了多个报文，但中间有报文出现了丢失或者乱序，这会使得接收端的窗口产生空洞(即在未收到序号较小的报文时，先收到序号较大的报文)。这种情况可能预示着链路发生了拥塞，因此，这种时候最好能让接收方 Echo 稍早时候的 TSval ，而不是序号最大报文的 TSval , 这样使得发送端估算的 RTT 能偏大，也就是发送报文更保守(conservative)，有利于减小拥塞。

#### 3.接收端序号空洞被填上

接收端序号空洞被填上意味着 **1. 乱序的报文姗姗来迟**; 或者 **2.收到了发送端重传报文**；

无论哪种情况，这都是反映了网络的真实情况，因此这个时候接收端应该 Echo 这个报文的 TSval。


#### 算法

RFC 中设计了一个算法可以处理以上这些特殊场景，并且该算法也兼容正常应答的场景。

1.本地为每条 TCP 连接维护了两个 32-bit 的变量 TS.Recent 和 Last.ACK.sent。TS.recent 用来保存下一个填入 TSecr 的时间戳,当需要发送报文时，报文的 TSecr 始终从当前 TS.Recent 获得。
Last.ACK.sent  用来保存上一个应答的报文序号。在不启用 delay-ACK 时，它始终会与 RCV.NXT 保持同步；如果启用 delay-ACK，则每收到一个有序报文，RCV.NXT 会立即向后推进，而 Last.ACK.sent 不会，Last.ACK.sent 只会在真正 ACK 报文发送过后才更新。

2.如果收到的报文的时间戳大于 TS.Recent 并且报文没有造成空洞，则立即更新 TS.Recent
 
```
 if SEG.TSval >= TS.Recent and SEG.SEQ <= Last.ACK.sent
 then
    TS.Recent = SEG.TSval  
```

来看看这个算法对上面提到的特殊情况的处理

- case 1, 接收端启用了 Delay ACK, 报文 A-B-C 按序到达:
```
                                                  TS.Recent   Last.ACK.sent  RCV.NXT
                <A, TSval=1> ------------------->                             
                                                      1            A          A->B
                <B, TSval=2> ------------------->
                                                      1            A          B->C
                <C, TSval=3> -------------------> 
                                                      1            A          C->D
                         <---- <ACK(C), TSecr=1>                   
                (etc.)                                1           A->D         D
```

启用了 Delay ACK  => Last.ACK.sent 直到发出 ACK 才更新 =>  TS.Recent 一直未更新 => ACK(C)的 TSecr=1

- case 2, 未启用 Delay ACK，但有报文乱序到达了:
```
                                                  TS.Recent   Last.ACK.sent  RCV.NXT
                <A, TSval=1> ------------------->
                                                      1           A           A->B 
                         <---- <ACK(A), TSecr=1>
                                                      1          A->B          B
                <C, TSval=3> ------------------->
                                                      1           B            B                         
                         <---- <ACK(A), TSecr=1>
                                                      1           B            B
                <B, TSval=2> ------------------->
                                                     1->2         B           B->D           
                         <---- <ACK(C), TSecr=2>
                                                      2          B->D          D
                <E, TSval=5> ------------------->
                                                      2            D           D
                         <---- <ACK(C), TSecr=2>
                                                      2            D           D
                <D, TSval=4> ------------------->
                                                     2->4          D          D->F       
                         <---- <ACK(E), TSecr=4>
                                                      4           D->F         F
                (etc.)
```
报文 C 到达, 由于造成了接收端序号空洞，因此不会更新 TS.Recent,
报文 B 到达, 填上了空洞，因此 TS.Recent 更新.


### PAWS

PAWS 的全称是 Protection Against Wrapped Sequences，即防止序号回绕。它的本质实际上是**利用 TCP Timestamp 选项的单调递增特性来识别老旧的报文**，防止这些老旧报文的干扰。

我们知道 TSval 是发送端发送该报文的时间戳，越晚发出的报文显然时间戳应该越大, 因此如果 SEG.TSval < TS.Recent, 则说明这个这个报文实际上是一个老旧的报文，接收应该将其丢弃。

假设发送端发送报文 SEG1 阻塞在网络中，然后重传了这个报文 SEG2。 显然  SEG1.TSval < SEG2.TSval。
接收端先收到 SEG2，于是更新 TS.Recent = SEG2.TSval，而后又收到了 SEG1。这个时候由于 SEG1.TSval < TS.Recent,
因此接收端需要丢弃 SEG1。

**那这如何跟防止序号回绕产生联系呢？**

答案是：在高速 TCP 连接的场景中，32-bit 的 TCP 序号可能很快就能用一个轮回，而 Timestamp 可以用来甄别报文的序号是否是属于上一个序号周期。

举个例子，假设发送端的序号从 1 开始计数，此时已经发送了 2^32 + 4999 字节的数据，当前接收端的 RCV.NXT = 5000, TS.Recent = 100, 如果收到一个报文 SEG.SEQ = 5000, SEG.LEN = 1000, TSval=70
如果不看时间戳，那么接收端会认为这个报文正好是预期的报文，但是这个报文实际上却是第 5000~5999 字节的报文。

**TCP Timestamp 实际上是扩充了 32-bit 的 TCP 报文序号**。

这里还有一个非常罕见的例外情况：TCP Timestamp 自身发生了回绕。
前面提到过，按照 1ms 的时间戳时钟计算，32-bit 的时间戳回绕一次的周期是 24.8 天，
因此如果一条 TCP 连接在 24.8 天之内都没有收到对端的数据，这个时候 TS.Recent 应该被视作失效状态，此时收到的第一个报文便不能使用 PAWS 进行检验了，从第二个开始才能重新使用。

#### TCP Timestamp 的另一面

TCP Timestamp 选项虽然能带来好处, 但并不是所有的 TCP 连接都会使用该选项，比如 Windows 系统就是默认不不启用该选项的，而 Linux 系统则是默认启用了该选项。
据 tcpm 的统计，在全球范围内，使用了 TCP Timestamp 的连接比例大概为 [60%~70%](https://mailarchive.ietf.org/arch/msg/tcpm/861z901kDYtHSWxDOsic_Ejpqz8/) 。

不支持 TCP Timestamp 的理由是该选项占用的报文长度太多了，它会占用 TCP 报文首部的 10 个字节，而且是每个报文都会有这种损耗。

Yoshifumi Nishida 曾在 2018 年提出过 PAWS 的[替代方案](https://tools.ietf.org/html/draft-nishida-tcpm-disabling-paws-00)

而站在 Linux 内核的角度，Yuchung Cheng 则列举了 TCP Timestamp 在内核中的应用场景。
```
1. RTT estimation on retransmission - this is crucial when a TCP is in
constant recovery mode such as encountering a traffic policer. Note
that the standard ACK time approach no longer works if any sequence
acked has been transmitted, thus RTO may stale during extended
recovery period.

2. Undo operations (i.e. TCP Eifel RFC4015) requires TCP timestamps -
being able to detect false reactions to losses is particularly key for
"loss-based" congestion control. Such false positives are common in
wireless or even wired networks due to all sorts of dark queues and L2
optimizations. While DSACK and F-RTO can also be used to detect false
recoveries - both of them take at least one more round-trip and/or
require new application data, while timestamp can detect false
retransmission instantly on the original ACK

3. Receiver buffer auto-tuning uses TCP ACKs' TS value to echo to
measure RTT from the receiver to tune the buffer size.

4. Potentially TCP timestamps being generated with known units, can be
used to gauge the data arrival rate. The measurement can be valuable
for congestion control b/c it's not subject to hectic ACK delays in
modern networks. I know folks at NetFlix is doing some deep research
into this aspect.
```

