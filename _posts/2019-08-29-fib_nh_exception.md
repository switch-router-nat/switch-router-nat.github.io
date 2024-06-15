---
layout    : post
title     : "FIB nexthop Exception是什么"
date      : 2019-08-29
lastupdate: 2019-08-29
categories: Network(Kernel)
---

<p align="center"><img src="/assets/img/fib_nh_exception/exception.JPG"></p>

## 理论

3.6版本内核移除了FIB查询前的路由缓存，取而代之的是下一跳缓存，这在[路由缓存的前世今生](https://switch-router.gitee.io/blog/routecache/) 中已经说过了。本文要说的是在该版本中引入的另一个概念：`FIB Nexthop Exception`，用于记录下一跳的例外情形。

它有什么用呢？

我们知道，FIB表项来源于路由配置(用户手动或者路由进程计算)，说到底，它们都来源于用户空间的设置。这些表项基本是稳定的。但内核实际发包时，还有两个情况需要考虑

- 收到过ICMP REDIRECT报文，表示之前发送的报文绕路了，之后的报文应该修改报文的下一跳。
- 收到过ICMP FRAGNEEDED报文，表示之前的报文太大了，路径上的一些设备不接受，需要源端进行分片。

这两种情况是针对单一目的地址的，什么意思呢？已PMTU为例，在下面的网络拓扑中，我在主机A上配置了下面一条路由

<p align="center"><img src="/assets/img/fib_nh_exception/topo.JPG"></p>
```
ip route add 4.4.0.0/16 via 4.4.4.4
```
意思是向所有目的地址在4.4.0.0/16的主机发包，下一跳都走192.168.99.1。

当A发送一个长度为1500的IP报文给C时 ，中间的一台网络设备B觉得这个报文太大了，因此它向A发送ICMP FRAGNEEDED报文,说我只能转发1300以下的报文，请将报文分片。A收到该报文后怎么办呢？总不能以后命中这条路由的报文全部按1300发送吧，因为并不是所有报文的路径都会包含B。

这时FIB Nexthop Exception就派上用场了，他可以记录下这个例外情况。当发送报文命中这条路由时，如果目的地址不是C，那么按1500进行分片，如果是C，则按1300进行分片。

## 实现

内核中使用`fib_nh_exception`表示这种例外表项

```c
(include/net/ip_fib.h)

struct fib_nh_exception {
	struct fib_nh_exception __rcu	*fnhe_next;  /*  冲突链上的下个fib_nh_exception结构 */
	__be32				fnhe_daddr;              /*  例外的目标地址                     */
	u32				  fnhe_pmtu;                 /*  收到的ICMP FRAGNEEDED通告的PMTU    */
	__be32				fnhe_gw;                 /*  收到的ICMP REDIRECT通告的网关      */         
	unsigned long			fnhe_expires;        /*  该例外表项的过期时间                */
	struct rtable __rcu		*fnhe_rth;           /*  关联的路由缓存                     */
	unsigned long			fnhe_stamp;
};
```

每一个下一跳结构`fib_nh`上有一个指针指向`fnhe_hash_bucket`哈希桶的指针:

```
struct fib_nh {
	/* code omitted */
	struct fnhe_hash_bucket	*nh_exceptions;
};
```

哈希桶在`update_or_create_fnhe`中创建，每个哈希桶包含2048条冲突链，每条冲突链可以存5个`fib_nh_exception`

以PMTU为例，在收到网络设备返回的ICMP FRAGNEEDED报文后，会调用下列函数将通告的pmtu值记录到`fib_nh_exception`上(也会记录到绑定的路由缓存`rtable`上)

```c
static void __ip_rt_update_pmtu(struct rtable *rt, struct flowi4 *fl4, u32 mtu)
{
    /* */
	if (fib_lookup(dev_net(dst->dev), fl4, &res) == 0) {
		struct fib_nh *nh = &FIB_RES_NH(res);

		update_or_create_fnhe(nh, fl4->daddr, 0, mtu,
				      jiffies + ip_rt_mtu_expires);
	}
}
```

而在发包流程查询FIB之后，会首先看是否存在以目标地址为KEY的例外表项，如果有，就使用其绑定的路由缓存，如果没有就使用下一跳上的缓存
```c
static struct rtable *__mkroute_output(const struct fib_result *res,
				       const struct flowi4 *fl4, int orig_oif,
				       struct net_device *dev_out,
				       unsigned int flags)
{
    /* code omitted */
    
    if (fi) {
		struct rtable __rcu prth;
		struct fib_nh *nh = &FIB_RES_NH(*res);

		fnhe = find_exception(nh, fl4->daddr);      //  查找 fl4->daddr 是否存在 fib_nh_exception
		if (fnhe)
			prth = &fnhe->fnhe_rth;                  // 如果有，直接使用其绑定的路由缓存
		else {
			if (unlikely(fl4->flowi4_flags &
				     FLOWI_FLAG_KNOWN_NH &&
				     !(nh->nh_gw &&
				       nh->nh_scope == RT_SCOPE_LINK))) {
				do_cache = false;
				goto add;
			}
			prth = __this_cpu_ptr(nh->nh_pcpu_rth_output);   // 如果没有，使用下一跳上缓存的路由缓存
		}
		rth = rcu_dereference(*prth);
		if (rt_cache_valid(rth)) {
			dst_hold(&rth->dst);
			return rth;
		}
	}
}
```





