---
layout    : post
title     : "TCP listen套接字的查找的变化"
date      : 2019-09-29
lastupdate: 2019-09-29
categories: Network(Kernel)
---
内核TCP在收到SYN报文时，会根据报文的目的IP和Port，在本地匹配处于LISTEN状态的套接字进行握手过程。

### 4.17版本以前的listen套接字查找

> The current listener hashtable is hashed by port only. When a process is listening at many IP addresses with the same port (e.g.[IP1]:443, [IP2]:443... [IPN]:443), the inet[6]_lookup_listener() performance is degraded to a link list.  It is prone to syn attack.

4.17版本之前，TCP的listener socket是按`port`进行hash，然后插入到对应的冲突链表中的。这就使得如果很多个listen套接字都侦听同一个port，就会使得链表拉得比较长, 这种情况在3.9版本引入`REUSEPORT`之后更加严重

<p align="center"><img src="https://s2.ax1x.com/2019/09/29/uGgart.png"></p>
举个栗子，主机上启动了6个listener,它们都侦听21端口，因此被放到同一条链表上(其中`sk_B`使用了`REUSEPORT`)。如果此时收到一个目标位`1.1.1.4:21`的SYN连接请求,内核在查找listenr的时候，始终会从头开始遍历到尾，直到找到匹配的`sk_D`。

### 4.17版本：在两个hashtable中查找

4.17版本增加了一个新的hashtable(`lhash2`)来组织listen套接字，这个`lhash2`是按`port+addr`作为key进行hash的，而原来按`port`进行hash的hashtable保持不变。换句话说，同一个listen套接字会同时放到两个hashtable中(例外情况是，如果它绑定的本地地址是0.0.0.0,则只会放到原来的hashtable中)

`lhash2`增加了addr作为key，也就增加hash的随机性。还是以上面的例子为例，此时，原来的`sk_A~C`可能就被hash到其他冲突链了,当然与此同时，也有可能有原来在其他冲突链上的`sk_E`被hash到`lhash2[0]`这条冲突链。

<p align="center"><img src="https://s2.ax1x.com/2019/09/29/uGgtxA.png"></p>
因此在listen套接字的查找时，内核会根据SYN报文中的`port+addr`，同时计算出满足条件的套接字应该在两个hashtable中所属的链表，然后比较这两个链表的长度，如果在1st链表长度不长或者小于2nd链表的长度，则还是以原来的方式，在1st链表中进行查找，否则就在2nd链表中进行查找。

```diff
 				    struct inet_hashinfo *hashinfo,
 				    struct sk_buff *skb, int doff,
@@ -217,10 +306,42 @@ struct sock *__inet_lookup_listener(struct net *net,
 	unsigned int hash = inet_lhashfn(net, hnum);
 	struct inet_listen_hashbucket *ilb = &hashinfo->listening_hash[hash];
 	bool exact_dif = inet_exact_dif_match(net, skb);
+	struct inet_listen_hashbucket *ilb2;
 	struct sock *sk, *result = NULL;
 	int score, hiscore = 0;
+	unsigned int hash2;
 	u32 phash = 0;
 
+	if (ilb->count <= 10 || !hashinfo->lhash2)
+		goto port_lookup;
+
+	/* Too many sk in the ilb bucket (which is hashed by port alone).
+	 * Try lhash2 (which is hashed by port and addr) instead.
+	 */
+
+	hash2 = ipv4_portaddr_hash(net, daddr, hnum);
+	ilb2 = inet_lhash2_bucket(hashinfo, hash2);
+	if (ilb2->count > ilb->count)
+		goto port_lookup;
+
+	result = inet_lhash2_lookup(net, ilb2, skb, doff,
+				    saddr, sport, daddr, hnum,
+				    dif, sdif);
+	if (result)
+		return result;
+
+	/* Lookup lhash2 with INADDR_ANY */
+
+	hash2 = ipv4_portaddr_hash(net, htonl(INADDR_ANY), hnum);
+	ilb2 = inet_lhash2_bucket(hashinfo, hash2);
+	if (ilb2->count > ilb->count)
+		goto port_lookup;
+
+	return inet_lhash2_lookup(net, ilb2, skb, doff,
+				  saddr, sport, daddr, hnum,
+				  dif, sdif);
+
+port_lookup:
 	sk_for_each_rcu(sk, &ilb->head) {
 		score = compute_score(sk, net, hnum, daddr,
 				      dif, sdif, exact_dif);
```

### 5.0版本：只在2nd hashtable中查找

内核在5.0版本又将查找方式改为了只在2nd hashtable中进行查找。这样修改的原因是按原来的查找方式，如果选择了在1st hashtable中进行查找，可能发生在通配地址(0.0.0.0)和特定地址(比如1.1.1.1)都侦听同一个`Port`时，反而匹配上通配地址的listener的问题。这其实不是4.17版本的锅，而是在3.9版本引入`SO_PORTREUSE`就已经存在了！

来看看怎么回事：


<p align="center"><img src="https://s2.ax1x.com/2019/09/29/uGgUKI.png"></p>
设置了`SO_REUSEPORT`的`sk_A`和`sk_B`同时侦听21端口，如果`sk_A`是后启动，那么它将添加到链表头，这样当收到一个`1.1.1.2:21`的报文时，内核会发现`sk_A`就已经匹配了，它就不会再去尝试匹配更精确的`sk_B`！这显然不好，要知道在`SO_REUSEPORT`进入内核之前，内核会遍历整个链表，对每个套接字进行匹配程度打分(`compute_score`)。

5.0版本修改为只在2nd hashtable中进行查找，并且修改了`compute_score`的实现方式，如果侦听地址与报文的目的地址不相同，则直接算匹配失败。而在之前，通配地址是可以直接通过这项检查的。

查找方式的修改：
```diff
struct sock *__inet_lookup_listener(struct net *net,
 				    const __be32 daddr, const unsigned short hnum,
 				    const int dif, const int sdif)
 {
-	unsigned int hash = inet_lhashfn(net, hnum);
-	struct inet_listen_hashbucket *ilb = &hashinfo->listening_hash[hash];
-	bool exact_dif = inet_exact_dif_match(net, skb);
 	struct inet_listen_hashbucket *ilb2;
-	struct sock *sk, *result = NULL;
-	int score, hiscore = 0;
+	struct sock *result = NULL;
 	unsigned int hash2;
-	u32 phash = 0;
-
-	if (ilb->count <= 10 || !hashinfo->lhash2)
-		goto port_lookup;
-
-	/* Too many sk in the ilb bucket (which is hashed by port alone).
-	 * Try lhash2 (which is hashed by port and addr) instead.
-	 */
 
 	hash2 = ipv4_portaddr_hash(net, daddr, hnum);
 	ilb2 = inet_lhash2_bucket(hashinfo, hash2);
-	if (ilb2->count > ilb->count)
-		goto port_lookup;
 
 	result = inet_lhash2_lookup(net, ilb2, skb, doff,
 				    saddr, sport, daddr, hnum,
@@ -335,34 +313,12 @@ struct sock *__inet_lookup_listener(struct net *net,
 		goto done;
 
 	/* Lookup lhash2 with INADDR_ANY */
-
 	hash2 = ipv4_portaddr_hash(net, htonl(INADDR_ANY), hnum);
 	ilb2 = inet_lhash2_bucket(hashinfo, hash2);
-	if (ilb2->count > ilb->count)
-		goto port_lookup;
 
 	result = inet_lhash2_lookup(net, ilb2, skb, doff,
-				    saddr, sport, daddr, hnum,
+				    saddr, sport, htonl(INADDR_ANY), hnum,
 				    dif, sdif);
-	goto done;
-
-port_lookup:
-	sk_for_each_rcu(sk, &ilb->head) {
-		score = compute_score(sk, net, hnum, daddr,
-				      dif, sdif, exact_dif);
-		if (score > hiscore) {
-			if (sk->sk_reuseport) {
-				phash = inet_ehashfn(net, daddr, hnum,
-						     saddr, sport);
-				result = reuseport_select_sock(sk, phash,
-							       skb, doff);
-				if (result)
-					goto done;
-			}
-			result = sk;
-			hiscore = score;
-		}
-	}
```


打分部分的修改
```diff
@@ -234,24 +234,16 @@ static inline int compute_score(struct sock *sk, struct net *net,
 				const int dif, const int sdif, bool exact_dif)
 {
 	int score = -1;
-	struct inet_sock *inet = inet_sk(sk);
-	bool dev_match;
 
-	if (net_eq(sock_net(sk), net) && inet->inet_num == hnum &&
+	if (net_eq(sock_net(sk), net) && sk->sk_num == hnum &&
 			!ipv6_only_sock(sk)) {
-		__be32 rcv_saddr = inet->inet_rcv_saddr;
-		score = sk->sk_family == PF_INET ? 2 : 1;
-		if (rcv_saddr) {
-			if (rcv_saddr != daddr)
-				return -1;
-			score += 4;
-		}
-		dev_match = inet_sk_bound_dev_eq(net, sk->sk_bound_dev_if,
-						 dif, sdif);
-		if (!dev_match)
+		if (sk->sk_rcv_saddr != daddr)
+			return -1;
+
+		if (!inet_sk_bound_dev_eq(net, sk->sk_bound_dev_if, dif, sdif))
 			return -1;
-		score += 4;
 
+		score = sk->sk_family == PF_INET ? 2 : 1;
 		if (sk->sk_incoming_cpu == raw_smp_processor_id())
 			score++;
 	}
```


### 附录:完整补丁

- [inet: Add a 2nd listener hashtable (port+addr) inet_connection_sock.h](https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/commit/include/net/inet_connection_sock.h?id=61b7c691c7317529375f90f0a81a331990b1ec1b)
- [inet: Add a 2nd listener hashtable (port+addr) inet_hashtables.h](https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/commit/include/net/inet_hashtables.h?id=61b7c691c7317529375f90f0a81a331990b1ec1b)
- [inet: Add a 2nd listener hashtable (port+addr) inet_hashtables.c](https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/commit/net/ipv4/inet_hashtables.c?id=61b7c691c7317529375f90f0a81a331990b1ec1b)
- [net: tcp: prefer listeners bound to an address inet_hashtables.c](https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/commit/net/ipv4/inet_hashtables.c?id=d9fbc7f6431fc0e5c0ddedf72206d7c5175c5c9a)
