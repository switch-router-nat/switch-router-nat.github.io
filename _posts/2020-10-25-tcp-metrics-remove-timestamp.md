---
layout    : post
title     : "TCP Metrics--remove per-destination timestamp cache"
date      : 2020-10-25
lastupdate: 2020-10-25
categories: Network(Kernel)
---

<p align="center"><img src="/assets/img/public/tcp.png"></p>

2017年3月，内核主线将`TCP Metrics`表项中的时间戳缓存，补丁详见[patch---tcp: remove per-destination timestamp cache](https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/commit/net/ipv4/tcp_metrics.c?id=d82bae12dc38d79a2b77473f5eb0612a3d69c55b)

```diff
 struct tcp_metrics_block {
 	struct inetpeer_addr		tcpm_saddr;
 	struct inetpeer_addr		tcpm_daddr;
 	unsigned long			tcpm_stamp;
-	u32				tcpm_ts;
-	u32				tcpm_ts_stamp;
 	u32				tcpm_lock;
 	u32				tcpm_vals[TCP_METRIC_MAX_KERNEL + 1];
```

与之一起修改的，还有[tcp: remove tcp_tw_recycle](https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/commit/net/ipv4/tcp_input.c?id=4396e46187ca5070219b81773c4e65088dac50cc)。`tcp_tw_recycle`机制是用于内核快速回收**TIME_WAIT**状态的套接字。但是当网络中存在NAT设备时，该机制反而可能会导致NAT设备背后的客户端难以连接上服务器。

这样的问题实在太多了！网络上随便一搜就是
[No response to some SYN packets when timestamps are enabled](https://serverfault.com/questions/583488/no-response-to-some-syn-packets-when-timestamps-are-enabled?)
[Why would a server not send a SYN/ACK packet in response to a SYN packet](https://serverfault.com/questions/235965/why-would-a-server-not-send-a-syn-ack-packet-in-response-to-a-syn-packet)
[why-does-my-linux-box-fail-to-send-syn-ack-until-after-eight-syns-have-arrived?](https://serverfault.com/questions/253358/why-does-my-linux-box-fail-to-send-syn-ack-until-after-eight-syns-have-arrived?)

导致这些问题的原因是服务器收到的**SYN报文中携带的时间戳早于之前已经收到的FIN报文的时间戳**，于是服务器认为该SYN报文是由于网络阻塞迟到的旧连接的SYN报文的重传，于是拒绝恢复SYN-ACK。出现这种情况的原因是传输链路上存在NAT设备。而缓存FIN报文时间戳的`TCP Metrics`是`Per-Destination`的，在有NAT的环境中，服务器看到的`Destination`是NAT设备，它看不到NAT设备背后还有多大的内部网络，内部网路的每台主机上无法保证SYN报文的时间戳递增。

当然，如果和我一样，不能升级内核, 那么不打开`tcp_tw_recycle`也是一个选择

>   tcp_tw_recycle (Boolean; default: disabled; since Linux 2.4) Enable fast recycling of TIME_WAIT sockets.  Enabling this option is not recommended for devices communicating with the general Internet or using NAT (Network Address Translation).Since some NAT gateways pass through IP timestamp values, one IP can appear to have non-increasing timestamps.  See RFC 1323 (PAWS), RFC 6191.


