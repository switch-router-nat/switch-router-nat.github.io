---
layout    : post
title     : "packetdrill--测试TCP协议栈行为的利器"
date      : 2019-05-15
lastupdate: 2019-05-15
categories: Network(others)
---

<p align="center"><img src="/assets/img/packetdrill/google.png"></p>

> 摘要：`packetdrill`是一个非常有用的用于测试网络协议栈的工具，由`Google`开发，它常用于对网络协议栈进行回归测试，确保新的功能不会影响原有功能。本文主要介绍其基本原理、安装、入门、测试脚本的编写方法。

## 1.简介
`packetdrill`是一个非常有用的用于测试网络协议栈的工具，由`Google`开发，它常用于对网络协议栈进行回归测试，确保新的功能不会影响原有功能。它支持**Linux**, **FreeBSD**, **OpenBSD**与**NetBSD**内核。它使用脚本化的语言编写测试语句，预测协议栈输出，官方也提供了许多测试脚本的例子。

## 2.原理

`packetdrill`的整体框架如下图所示

<p align="center"><img src="/assets/img/packetdrill/arch.png"></p>
`packetdrill`应用内部模拟了一个连接的`Remote端`和`Local端`。其中`Remote端`用作远端发送到本机报文的通道，我们可以在`packetdrill`应用内向`tun`设备写入`IP`报文，对内核协议栈来来，这相当于从远端收到了这个`IP`报文，再经过路由，这个报文会上送协议栈。反过来说，内核协议栈的向`Remote端`发送的报文会通过这个`tun`设备回到`packetdrill`应用，这时，我们可以通过比对其输入，验证协议栈的功能正确性。

脚本文件是以`.pkt`为后缀的文件，`packetdrill`启动后读取该文件，`脚本解析器`将每一行脚本语句其解析为运行时`event`，`脚本运行机`依次执行每个`event`。

## 3.安装

`packetdrill`依赖的 **package**: `gcc`、`python`、`flex`、`bison`

从[官方github](https://github.com/google/packetdrill)下载源代码后,编译即可

```bash
> ./configure
> make
```

## 4.入门

### 执行一个测试脚本

```bash
> ./packetdrill  tests/linux/fast_retransmit/fr-4pkt-sack-linux.pkt
> 
```

如果没有任何输出，就表示脚本测试通过了:)，否则，它会提示哪一行脚本不满足预期以及错误原因分别是什么

比如在我的机器上(内核版本4.4.0)执行下面脚本的时候出现了错误：

```bash
> ./packetdrill  tests/linux/listen/listen-incoming-ack.pkt 
tests/linux/listen/listen-incoming-ack.pkt:17: error handling packet: bad value outbound TCP option 3
script packet:  0.200000 S. 0:0(0) ack 1 <mss 1460,nop,nop,sackOK,nop,wscale 6>
actual packet:  0.201014 S. 0:0(0) ack 1 win 29200 <mss 1460,nop,nop,sackOK,nop,wscale 7>
>
```
它表示在执行到脚本第`17`行的时候出现了错误，脚本中`remote端`预期收到的`SYNACK`报文中`wscale=6`，但实际收到的报文中`wscale=7`。

出现这种错误的原因是脚本适合的内核版本的协议栈实现与我本机版本中的不一致！内核版本不一致，协议栈的某些实现就不一致！遇到这种情况时，可以简单的修改脚本以适应我们自己使用的内核版本。

## 5. 脚本语言

`packetdrill`并没有使用某一种现成的脚本语言，它的脚本有一些`tcpdump`影子，又有一些`socket`编程的踪迹

```bash
// Test behavior when a listener gets an incoming packet that has
// the ACK bit set but not the SYN bit set.

0.000 socket(..., SOCK_STREAM, IPPROTO_TCP) = 3
0.000 setsockopt(3, SOL_SOCKET, SO_REUSEADDR, [1], 4) = 0
0.000 bind(3, ..., ...) = 0
0.000 listen(3, 1) = 0

0.100 < . 0:0(0) win 32792 <mss 1000,sackOK,nop,nop,nop,wscale 7>
0.100 > R 0:0(0) win 0

// Now make sure that when a valid SYN arrives shortly thereafter
// (with the same address 4-tuple) we can still successfully establish
// a connection.

0.200 < S 0:0(0) win 32792 <mss 1000,sackOK,nop,nop,nop,wscale 7>
0.200 > S. 0:0(0) ack 1 <mss 1460,nop,nop,sackOK,nop,wscale 6>

0.300 < . 1:1(0) ack 1 win 320
0.300 accept(3, ..., ...) = 4
```

我觉得这种脚本最好的学习方法就是学习官方的例子了，在例子上依葫芦画瓢就可以构造出自己需要的脚本了！实在有疑惑还可以稍微翻翻代码！

### 时间戳

脚本以每行为单位，每一行都是`时间戳 + 语句`的形式。时间戳表示这条语句执行的时间，`packetdrill`支持`绝对时间`和`相对时间`两种格式.

```
// 相对时间，上一条脚本0.1秒后
+.1  setsockopt(3, SOL_SOCKET, SO_REUSEADDR, [1], 4) = 0

// 绝对时间，脚本开始运行后0.2秒后
0.200 setsockopt(3, SOL_SOCKET, SO_REUSEADDR, [1], 4) = 0
```

### 向协议栈注入报文

时间戳后跟着`<`符号的语句表示从`remote端`向协议栈注入报文，默认后面跟的是`TCP`报文的内容(当然也可以接其他协议，但协议栈的复杂之处大多在`TCP`)

```
//注入一个SYN报文(S表示SYN)，起始序号和结束序号为0，数据长度为0，通告窗口大小为32792，携带了mss、sack和wscale的选项
0.200 < S 0:0(0) win 32792 <mss 1000,sackOK,nop,nop,nop,wscale 7>
```

### 从协议栈接收报文

时间戳后跟着`>`符号的语句表示`remote端`预期从协议栈接收报文。这里的预期接收时间是一个范围[`ts-tolerance`,`ts+tolerance`]，容忍时间`tolerance`默认是`4`毫秒(可以通过运行参数改变)

```
// 预期收到一个SYNACK报文(.表示ACK) ACK序号是1 ，携带了mss、sack和wscale的选项
0.200 > S. 0:0(0) ack 1 <mss 1460,nop,nop,sackOK,nop,wscale 6>
```

### 系统调用

上面的报文语句是站在`remote端`的角度的，系统调用是站在`local端`看的，`packetdrill`支持以下的系统调用

```
struct system_call_entry system_call_table[] = {
	{"socket",     syscall_socket},
	{"bind",       syscall_bind},
	{"listen",     syscall_listen},
	{"accept",     syscall_accept},
	{"connect",    syscall_connect},
	{"read",       syscall_read},
	{"readv",      syscall_readv},
	{"recv",       syscall_recv},
	{"recvfrom",   syscall_recvfrom},
	{"recvmsg",    syscall_recvmsg},
	{"write",      syscall_write},
	{"writev",     syscall_writev},
	{"send",       syscall_send},
	{"sendto",     syscall_sendto},
	{"sendmsg",    syscall_sendmsg},
	{"fcntl",      syscall_fcntl},
	{"ioctl",      syscall_ioctl},
	{"close",      syscall_close},
	{"shutdown",   syscall_shutdown},
	{"getsockopt", syscall_getsockopt},
	{"setsockopt", syscall_setsockopt},
	{"poll",       syscall_poll},
	{"cap_set",    syscall_cap_set},
	{"open",       syscall_open},
	{"sendfile",   syscall_sendfile},
	{"epoll_create", syscall_epoll_create},
	{"epoll_ctl",    syscall_epoll_ctl},
	{"epoll_wait",   syscall_epoll_wait},
	{"pipe",         syscall_pipe},
	{"splice",       syscall_splice},
};
```

与我们习惯的的系统调用不一样的地方是，`packetdrill`中的系统调用中有一些参数是不可以更改的，我们需要填写`...`(脚本运行机会帮我们填上)，另外还要设置其返回值。

```bash
// socket系统调用,返回 fd = 3，这里的3只在脚本范围中有效，运行时返回的描述符的值由框架内部维护，框架会维护它们的对应关系
0.000 socket(..., SOCK_STREAM, IPPROTO_TCP) = 3

// setsockopt系统调用，第三个参数[1]表示一个指向数值1的指针
0.000 setsockopt(3, SOL_SOCKET, SO_REUSEADDR, [1], 4) = 0

// listen系统调用，后两个参数由框架确定
0.000 bind(3, ..., ...) = 0
0.000 listen(3, 1) = 0
```

### assert

有时我们还需要窥测`TCP`运行时的更多信息，比如双方协商的`MSS`是多少，当前的窗口大小`cwnd`是多少，慢启动阈值`ssthresh`是多少。这时我们可以使用`assert`语句来预期其状态

```bash
// 预期此时的状态信息
0.300 %{
assert tcpi_reordering == 3
assert tcpi_unacked == 10
assert tcpi_sacked ==  1
}%
```

`packetdrill`支持预期`TCP`信息如下：

```c
/* packetdrill/gtests/net/packetdrill/tcp.h */
struct _tcp_info {
	__u8	tcpi_state;
	__u8	tcpi_ca_state;
	__u8	tcpi_retransmits;
	__u8	tcpi_probes;
	__u8	tcpi_backoff;
	__u8	tcpi_options;
	__u8	tcpi_snd_wscale:4, tcpi_rcv_wscale:4;
	__u8	tcpi_delivery_rate_app_limited:1;

	__u32	tcpi_rto;
	__u32	tcpi_ato;
	__u32	tcpi_snd_mss;
	__u32	tcpi_rcv_mss;

	__u32	tcpi_unacked;
	__u32	tcpi_sacked;
	__u32	tcpi_lost;
	__u32	tcpi_retrans;
	__u32	tcpi_fackets;

	/* Times. */
	__u32	tcpi_last_data_sent;
	__u32	tcpi_last_ack_sent;     /* Not remembered, sorry. */
	__u32	tcpi_last_data_recv;
	__u32	tcpi_last_ack_recv;

	/* Metrics. */
	__u32	tcpi_pmtu;
	__u32	tcpi_rcv_ssthresh;
	__u32	tcpi_rtt;
	__u32	tcpi_rttvar;
	__u32	tcpi_snd_ssthresh;
	__u32	tcpi_snd_cwnd;
	__u32	tcpi_advmss;
	__u32	tcpi_reordering;

	__u32	tcpi_rcv_rtt;
	__u32	tcpi_rcv_space;

	__u32	tcpi_total_retrans;

	__u64	tcpi_pacing_rate;
	__u64	tcpi_max_pacing_rate;
	__u64	tcpi_bytes_acked;    /* RFC4898 tcpEStatsAppHCThruOctetsAcked */
	__u64	tcpi_bytes_received; /* RFC4898 tcpEStatsAppHCThruOctetsReceived */
	__u32	tcpi_segs_out;	     /* RFC4898 tcpEStatsPerfSegsOut */
	__u32	tcpi_segs_in;	     /* RFC4898 tcpEStatsPerfSegsIn */

	__u32	tcpi_notsent_bytes;
	__u32	tcpi_min_rtt;
	__u32	tcpi_data_segs_in;	/* RFC4898 tcpEStatsDataSegsIn */
	__u32	tcpi_data_segs_out;	/* RFC4898 tcpEStatsDataSegsOut */
	__u64   tcpi_delivery_rate;

	__u64	tcpi_busy_time;      /* Time (usec) busy sending data */
	__u64	tcpi_rwnd_limited;   /* Time (usec) limited by receive window */
	__u64	tcpi_sndbuf_limited; /* Time (usec) limited by send buffer */
};

```

需要特别注意是，在使用`assert`时，我们要确定`struct _tcp_info`结构在`packetdrill`中和当前内核中的定义一致，否则也会报错！

(完)
