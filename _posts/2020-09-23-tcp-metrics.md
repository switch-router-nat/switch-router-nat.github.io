---
layout    : post
title     : "内核TCP Metrics框架"
date      : 2020-09-23
lastupdate: 2020-09-23
categories: Network(Kernel)
---

<p align="center"><img src="/assets/img/public/tcp.png"></p>

TCP是一个复杂的协议，这种复杂来源于对报文传输的可靠性承诺。**对每条TCP连接来说**，除了有独立的状态机、定时器之外，还有拥塞控制相关的一些运行变量，比如**RTT**、**CWND**、**SSTHRESH**等，这些运行参数同样也是每连接(`Per-Connection`)的

`Per-Connection`意味着每条连接的这些参数互不影响，这是理所应当的！但是，想想这个情景：A与B之间已经建立了一条稳定的TCP连接，此时若新建一条新的连接，它的参数该如何设置呢？显然，和原连接保持一致是个快速达到稳定的办法。这就好比一个人要去一个陌生的地方，却不知道该选择哪种交通工具，也不知道该预估多少时间，对他来说，汲取去过的人的经验总是一条捷径。

这就是Linux内核中`TCP Metrics`框架的作用，**它可以为后续的连接提供指导**。当主机之间需要频繁**建立**和**拆除**TCP连接时，它带来的好处更加明显。

`TCP Metrics`显然不能是`Per-Connection`的，而应该是`Per-Host`的。也就是说，`TCP Metrics`表项应该是基于`<源IP,目的IP>`二元组的。从一台主机的角度，到达另一个特定地址主机的网络链路状况应该是被两台主机之间的所有连接所**共享**的。

内核使用`tcp_metrics_block`表示一条`Metrics`表项，这些表项根据`<源IP,目的IP>`组织在`tcp_metrics_hash`冲突链表表中，记录的值保存在内部`tcpm_vals`数组
```c
struct tcp_metrics_block {
	struct tcp_metrics_block __rcu	*tcpm_next;
	struct inetpeer_addr		tcpm_saddr;
	struct inetpeer_addr		tcpm_daddr;
	......
	u32				tcpm_vals[TCP_METRIC_MAX_KERNEL + 1];
    ......
};
```

当新建TCP连接时，内核使用下面的接口来为TCP套接字设置`TCP Metrics`指导下的参数
```c
void tcp_init_metrics(struct sock *sk)
```

当某条TCP连接收的运行参数发生变化时，比如重新计算RTT了，内核会使用下面的接口来更新它对应的`TCP Metrics`表项。切记，`TCP Metrics`表项是`Per-Host`的，因此，多条TCP连接的套接字可能会更新同一条表项。 
```
void tcp_update_metrics(struct sock *sk)
```

> 内核同样提供[ip-tcp_metrics](https://www.systutorials.com/docs/linux/man/8-ip-tcp_metrics/)命令查看主机上的`TCP Metrics`表项.






