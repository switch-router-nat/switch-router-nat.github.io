---
layout    : post
title     : "Linux 路由缓存的前世今生"
date      : 2019-08-25 
lastupdate: 2019-08-25 
categories: Network(Kernel)
---

<p align="center"><img src="/assets/img/routecache/squirrel cache.jpg"></p>
**3.6**版本一定算得上是**Linux**网络子系统中一个特别的版本, 这个版本(补丁[patch](https://git.kernel.org/pub/scm/linux/kernel/git/davem/net-next.git/commit/?id=89aef8921bfbac22f00e04f8450f6e447db13e42))移除了查找**FIB**之前的缓存查找。本文就来谈谈路由缓存的前世今生。


## 几个基本概念

为了让本文的阅读曲线更加平缓我决定还是将本文涉及的一些术语作个说明。

`路由`：将**skb**按照**规则**送到该去的地方，这个地方可能是本机，也可能是局域网中的其他主机,或者更远的主机。从这个角度来说，它一个**动词**。那么**路由**发生在哪个时候呢？ 我们知道**路由**是网络层(**L3**)的概念,接收方向，它需要决定收到的**skb**是应该**上送本机**还是**转发**,发送方向，它需要决定**skb**从哪个网络接口发出。下图原本是描述`Netfilter`在内核中的钩子位置的,但我觉得用来说明**路由**的位置也是比较合适的。

<p align="center"><img src="/assets/img/routecache/forward.PNG"></p>
与此同时，**路由**也可以特指上面所说的**规则**,这是**名词**的用法。**路由**从哪来？ 一般来说有三个来源：1. 用户主动配置；2.内核生成； 3. 其他一些路由协议进程(`OSPF`、`BGP`)生成。普通主机上可能没有最后一种，所以，为了理解方便，你可以将**路由**就理解为你用`route`命令看到的内容。

```
[root@tristan]# route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
192.168.99.0    0.0.0.0         255.255.255.0   U     0      0        0 eth0
192.168.98.42   192.168.99.1    255.255.255.255 UGH   0      0        0 eth0
127.0.0.0       0.0.0.0         255.0.0.0       U     0      0        0 lo
0.0.0.0         192.168.99.254  0.0.0.0         UG    0      0        0 eth0
```

`FIB`：全称是(**F**orwarding **I**nformation **B**ase)，翻译过来就是**转发信息表**。**FIB**是内核**skb**路由过程的数据库，或者说**内核会将路由翻译成FIB中的表项**。我们习惯说的**查询路由**，对于内核来说，应该叫**查询FIB**。

## 3.6版本以前的路由缓存

**缓存**无处不在。现代计算机系统中，**Cache**是**CPU**与内存间存在一种容量较小但速度很高的存储器，用来存放`CPU`刚使用过或最近使用的数据。**路由缓存就是基于这种思想的软件实现**。内核查询**FIB**前，固定先查询**cache**中的记录，如果**cache**命中(**hit**)，那就直接用就好了，不必查询**FIB**。如果没有命中(**miss**), 就回过头来查询**FIB**，最终将结果保存到**cache**，以便下次不再需要需要查询**FIB**。

缓存是精确匹配的, 每一条缓存表项记录了匹配的源地址和目的地址、接收发送的`dev`，以及与内核邻居系统(**L2**层)的联系(`negghbour`)
`FIB`中存储的也就是路由信息，它常常是范围匹配的，比如像`ip route 1.2.3.0/24 dev eth0`这样的网段路由。

下图是**3.6**版本以前的本机发送**skb**的路由过程....

<p align="center"><img src="/assets/img/routecache/xmit_route_process.JPG"></p>

看上去的确可能能提高性能! 只要**cache**命中率足够高。要获得高的**cache**命中率有以下两个途径：1. 存储更多的表项; 2.存储更容易命中的表项

缓存中存放的表项越多，那么目标报文与表项匹配的可能性越大。但是**cache**又不能无限制地增大，**cache**本身占用内存是一回事，更重要的是越多的表项会导致查询**cache**本身变慢。使用**cache**的目的是为了加速，如果不能加速，那要这劳什子有有什么用呢？

前面说了，**cache**的特点决定了它只能做**精确匹配**。也就是说，只有目标数据报文与**cache**中的表项完全一致，才算匹配成功。最简单的**cache**查找过程应该是下面这样：遍历**cache**中的所有表项，直到遇到匹配的表项跳出循环。

```
foreach entry in cache:
then
    if entry match skb
    then
        /* 条件匹配，将缓存表项中记录的结果设置到skb上 */
        skb->dst <= entry->dst
        return
    endif
end
```
显然，**cache**表项的数目越多，那么查找的过程就越长! 当然，内核不会这么蠢地将所有**cache**拉成一个线，而是使用**hash**桶，看上去应该是这么一个结构。

<p align="center"><img src="/assets/img/routecache/cachebucket.JPG"></p>
内核首先根据目标报文的一些特征计算**hash**，找到对应的**hash**冲突链表。在链表上一个一个地进行比较遍历。 

为了避免**cache**表项过多，内核还会在**一定时机**下清除**过期**的表项。有两个这样的**时机**，其一是添加新的表项时，如果冲突链的表项过多，就删除一条已有的表项；其二是内核会启动一个专门的定时器周期性地**老化**一些表项.

获得更高的**cache**命中率的第二个途径是**存储更容易命中的表项**，什么是更容易命中的呢？ **那就是真正有效的报文**。遗憾的是，内核一点也不聪明：只要输入路由系统的报文不来离谱，它就会生成新的缓存表项。坏人正好可以利用这一点，不停地向主机发送垃圾报文，内核因此会不停地刷新**cache**。这样每个**skb**都会先在**cache表**中进行搜索，再查询**FIB表**，最后再创建新的**cache表项**，插入到**cache表**。这个过程中还会涉及为每一个新创建的**cache表项**绑定邻居，这又要查询一次`ARP`表。

要知道，一台主机上的路由表项可能有很多，特别是对于网络交换设备，由**OSPF**\**BGP**等路由协议动态下发的表项有上万条是很正常的事。而邻居节点却不可能达到这个数量。对于转发或者本机发送的**skb**来说，路由系统能帮它们找到下一跳**邻居**就足够了。

总结起来就是，**3.6**版本以前的这种路由缓存在**skb**地址稳定时的确可能提高性能。但这种根据**skb**内容决定的性能却是不可预测和不稳定的。

## 3.6版本以后的下一跳缓存

正如前面所说，**3.6**版本移除了**FIB**查找前的路由缓存。这意味着每一个接收发送的**skb**现在都必须要进行**FIB**查找了。这样的好处是现在查找路由的代价变得**稳定(consistent)**了。

路由缓存完全消失了吗? 并没有！在**3.6**以后的版本, 你还可以在内核代码中看到**dst_entry**。这是因为，**3.6**版本实际上是将**FIB**查找缓存到了**下一跳(fib_nh)**结构上，也就是**下一跳缓存**

为什么需要缓存下一跳呢？ 我们可以先来看下没有下一跳缓存的情况。以转发过程为例，相关的伪代码如下：
```
FORWARD:

fib_result = fib_lookup(skb)
dst_entry  = alloc_dst_entry(fib_result)
skb->dst = dst_entry;

skb->dst.output(skb)   
nexthop = rt_nexthop(skb->dst, ip_hdr(skb)->daddr)
neigh = ipv4_neigh_lookup(dev, nexthop)
dst_neigh_output(neigh,skb)
release_dst_entry(skb->dst)
```

内核利用**FIB**查询的结果申请**dst_entry**, 并设置到**skb**上，然后在发送过程中找到下一跳地址，继而查找到**邻居**结构(查询**ARP**)，然后邻居系统将报文发送出去，最后释放**dst_entry**。

下一跳缓存的作用就是尽量减少最初和最后的申请释放**dst_entry**，它将**dst_entry**缓存在下一跳结构(**fib_nh**)上。这和之前的路由缓存有什么区别吗？ 很大的区别！之前的路由缓存是以源IP和目的IP为KEY，有千万种可能性，而现在是和下一跳绑定在一起，一台设备没有那么多下一跳的可能。这就是下一跳缓存的意义！

### early demux 

`early demux`是在**skb**接收方向的加速方案。如前面所说，在取消了**FIB**查询前的路由缓存后，每个**skb**应该都需要查询**FIB**。而**early demux**是基于一种思想：如果一个**skb**是本机某个应用程序的套接字需要的，那么我们可以将路由的结果缓存在内核套接字结构上，这样下次同样的报文(四元组)到达后，我们可以在**FIB**查询前就将报文提交给上层，也就是**提前分流(early demux)**。

<p align="center"><img src="/assets/img/routecache/early_demux.JPG"></p>

## 总结

**3.6**版本将**FIB**查询之前的路由缓存移除了，取而代之的是下一跳缓存。

## REF

[Route cache removed](https://workshop.netfilter.org/2013/wiki/images/2/2a/DaveM_route_cache_removed_nfws2013.pdf)
[IPV4 route cache removed from >= 3.6 linux kernel](https://superuser.com/questions/503405/ipv4-route-cache-removed-from-3-6-linux-kernel)
[remove routing cache](http://vger.kernel.org/~davem/columbia2012.pdf)
[Linux3.5内核以后的路由下一跳缓存](https://blog.csdn.net/dog250/article/details/50809816)

 


