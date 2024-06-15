---
layout    : post
title     : "理解内核源端口选择--UDP"
date      : 2022-07-07
lastupdate: 2022-07-07
categories: Network(Kernel)
---

### 保存端口的数据结构

UDP 使用`udp_table`保存 udp socket 信息.

`udp_table` 包含两张表: `hash` 和 `hash2`, 前者仅根据 local port 进行哈希, 后者根据 local port, local address 进行哈希. 

<p align="center"><img src="/assets/img/udp-port-selection/pic1.png"></p>

关于`hash2` 的来历, 之前在[UDP bind](https://switch-router.gitee.io/blog/udp-bind/)文中有描述, 本文只需要关注`hash1`即可.

`hash1`的哈希函数`udp_hashfn` 是一个简单的取模函数:

```
static inline u32 udp_hashfn(const struct net *net, u32 num, u32 mask)
{
	return (num + net_hash_mix(net)) & mask;
}
```

其中 mask 为 `hash` 表的链表数量减一, 在我本地 mask 值为 2047 (2^11-1), 这说明同一条链上的 sock 的 local port 低 11 位是相同的.

### 端口选择的时机

UDP 选择源端口的可能的时机有三个: 1. bind() 2. connect() 3. sendto()

这三个时机有先后顺序: 一旦前一个时机已经完成端口选择, 后面将不再进行选择.

#### bind() 

部分 UDP client 应用可能使用`bind()`绑定本地地址和端口(可以填 0). 如果绑定到端口 0, 则内核会自己选择一个可用的端口. 其调用路径如下:

```
inet_bind
 |-- __inet_bind
   |-- (local addr bind)
   |-- sk->sk_prot->get_port
     |-- udp_v4_get_port
       |-- udp_lib_get_port
         |-- (local port bind/selection)
```

这种情况下, 在绑定/选择 local port 时, local addr 已经确定 (这会稍稍影响端口选择的逻辑)

#### connect()

部分 UDP client 应用可能使用`connect()`设置对端地址端口, 这样后续发送报文时就不必使用`sendto()`, 而只需使用`send()` 即可. 其调用路径如下:

```
sys_connect
 |-- __sys_connect_file
   |-- inet_dgram_connect
     |-- inet_autobind
       |-- sk->sk_prot->get_port
         |-- udp_v4_get_port
           |-- udp_lib_get_port
             |--  (local port selection)
     |-- (local addr selection)
```

这种情况下, 在选择 local port 时, local addr 尚未确定.

#### sendto()

如果 UDP client 应用没有既没有`bind()`, 也没有`connect()`,  那么在`sendto()`时, 内核才会进行源端口选择.

```
sys_sendto
 |-- sock_sendmsg
   |-- sock_sendmsg_nosec
     |-- inet_sendmsg
       |-- inet_send_prepare
         |-- inet_autobind
           |-- sk->sk_prot->get_port
             |-- udp_v4_get_port
                |-- udp_lib_get_port
                  |--  (local port selection)
     |-- (local addr selection)            

```

与上一种情况类似, 这种情况下, 在选择 local port 时, local addr 尚未确定.

### 如何端口选择

从上面的分析可以看出, 无论是什么时机触发, 内核最终都是在`udp_lib_get_port`中进行端口选择.

`udp_lib_get_port`的目标是从`sysctl net.ipv4.ip_local_port_range`的范围内找到一个与已使用端口不冲突的端口.

#### 端口冲突的判断条件

首先, 端口冲突判断的范围仅限 UDP 内部, 与 TCP 无关. 可以存在 UDP sock 和 TCP sock 保存相同的四元组.

其次, 在 UDP 内部, 冲突是指 local addr 与 local port 相同. 因此, 可以存在两个 UDP sock, 绑定到不同的 local addr 以及相同的 local port.

但需要注意, 如果在选择 local port 时, local addr 尚未确定 (对应`connect()`或`sendto()`两种情况), local addr 将视之与所有地址相同, 内核将避免选择的 local port 与所有潜在的 local port + local addr 冲突.

总结起来就是: 

```
已存在的 sock       正在进行选择的 sock         结果  
1.1.1.1:50000       127.0.0.1:50000           可用
1.1.1.1:50000       1.1.1.1:50000             冲突
1.1.1.1:50000       0.0.0.0:50000             冲突
```


#### 端口选择过程

UDP 端口选择的过程在历史上经过了几次变化.

第一次更新: [udp: Improve port randomization](https://github.com/torvalds/linux/commit/9088c5609584684149f3fb5b065aa7f18dcb03ff)
第二次更新: [udp: optimize bind(0) if many ports are in use](https://github.com/torvalds/linux/commit/98322f22eca889478045cf896b572250d03dc45f)

##### 第一次更新前--在最短链上选择

内核将遍历 `UDP_HTABLE_SIZE`(128) 条链，并找到其中最短的一条, 此时低 7 位就确定了, 接着在再将`ip_local_port_range`范围内每隔 `UDP_HTABLE_SIZE` 的端口作为候选端口，检查它与该链上其他 sock 是否冲突, 直到找到一个不冲突的端口.

可以看出, 这种方法的效率是比较低的, 因为每次选择都需要遍历所有链表. 

##### 第二次更新前--深度优先搜索

内核在`ip_local_port_range`范围内随机选择一个端口作为候选端口, 检查它是否与所属链上的其他 sock 冲突, 如果冲突, 就重新随机选择一个.

这种方法的缺点是一旦随机到比较长的链, 候选端口冲突的概率会较大, 这样就会反复进行加锁解锁.

> This is because we do about 28000 scans of very long chains (220 sockets per chain), with many spin_lock_bh()/spin_unlock_bh() calls.

##### 现在---bitmap

第二次更新最大变化是引入 bitmap, 修改了 `udp_lib_lport_inuse` 的实现. 

旧的 `udp_lib_lport_inuse` 只能检测候选端口是否与链上的其他 sock 冲突, 而新的 `udp_lib_lport_inuse` 会得出链上已占用端口组成的 bitmap  (这些端口的低 11 位是相同的).

然后内核即可从 bitmap 中找到一个未使用的端口. 

与更新前的区别在于: 更新前是先选一个答案再检查答案是否正确, 如果错误就重来; 更新后是先排除所有错误答案(已使用端口), 再选择. 效率熟高, 不言自明.

### 参考

[how-to-stop-running-out-of-ephemeral-ports-and-start-to-love-long-lived-connections](https://blog.cloudflare.com/how-to-stop-running-out-of-ephemeral-ports-and-start-to-love-long-lived-connections)
[再说说TCP和UDP源端口的确定](https://blog.csdn.net/dog250/article/details/82810226)