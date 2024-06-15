---
layout    : post
title     : "istio ambient 流量路径"
date      : 2023-10-15
lastupdate: 2023-10-15
categories: Others
---

<p align="center"><img src="/assets/img/public/ambient.png"></p>

本文用于梳理 istio ambient 模式下的流量路径. 

### 前置环境

首先使用 kind 搭建 ambient 模式的环境.

```
root@switch-router:/home/switch-router# kubectl get pod --all-namespaces -owide
NAMESPACE            NAME                                            READY   STATUS    RESTARTS   AGE   IP           NODE                    NOMINATED NODE   READINESS GATES
ambient              client-6db7cfc846-bwj8r                         1/1     Running   0          11h   10.244.1.4   ambient-worker          <none>           <none>
ambient              server-f44ffdf6f-dqgm2                          1/1     Running   0          11h   10.244.2.5   ambient-worker2         <none>           <none>
istio-system         ztunnel-g7mrp                                   1/1     Running   0          11h   10.244.1.2   ambient-worker          <none>           <none>
istio-system         ztunnel-w5lvt                                   1/1     Running   0          11h   10.244.2.3   ambient-worker2         <none>           <none>
```

其中运行在 ambient-worker2 上的 server pod 开启了 8080 端口. 

这里从 client pod (10.244.1.4) 去访问 server pod (10.244.2.5:8080)

```
root@switch-router:/home/switch-router# kubectl exec -it client-6db7cfc846-bwj8r -n ambient -- curl -v 10.244.2.5:8080
*   Trying 10.244.2.5:8080...
* Connected to 10.244.2.5 (10.244.2.5) port 8080 (#0)
> GET / HTTP/1.1
> Host: 10.244.2.5:8080
> User-Agent: curl/7.81.0
> Accept: */*
> 
* Mark bundle as not supporting multiuse
< HTTP/1.1 200 OK
< Date: Thu, 30 Nov 2023 14:22:27 GMT
< Content-Length: 0
< 
* Connection #0 to host 10.244.2.5 left intact
```

### 总览

虽然看上去只是一次简单的 http 访问, 但就和 sidecar 一样, 整个路径包含了**<mark>三段 TCP 连接</mark>**

<p align="center"><img src="/assets/img/istio-ambient-path/pic1.png"></p>

接下来将依次分析这 3 条 TCP 连接的数据路径

#### 1st TCP 路径

其中红色标号为 1st TCP 路径

<p align="center"><img src="/assets/img/istio-ambient-path/pic2.png"></p>

##### 1st TCP 阶段1 SYN 报文从 client Pod 发出

从 client Pod 发出的报文顺着 veth pair 到达 host

```
06:49:20.418254 IP 10.244.1.4.53696 > 10.244.2.5.8080: Flags [S], seq 3988947338, win 64240, options [mss 1460,sackOK,TS val 775716835 ecr 0,nop,wscale 7], length 0
```

##### 1st TCP 阶段2 策略路由转发 SYN 报文

进入协议栈，查询【附录1】. 命中 ztunnel-PREROUTING 链上的规则.  

```
ztunnel-PREROUTING -p tcp -m set --match-set ztunnel-pods-ips src -j MARK --set-xmark 0x100/0x100
```
SYN 报文的源 IP 在集合 ztunnel-pods-ips 内(【附录2】)，因此报文会被打上 0x100 标记. 

从【附录3】可知道, SYN 报文会在查询 table 101 后命中

```
default via 192.168.127.2 dev istioout
```
即报文转发到 istioout

##### 1st TCP 阶段3 UDP隧道封装 SYN 报文

```
14:49:20.418289 IP 10.244.1.4.53696 > 10.244.2.5.8080: Flags [S], seq 3988947338, win 64240, options [mss 1460,sackOK,TS val 775716835 ecr 0,nop,wscale 7], length 0
```

istioout 是一个 geneve tunnel 设备，从该设备发出的报文会在报文外添加上一个ip头、udp头和 geneve 头. 其中目的地址变为隧道的 remote 地址 10.244.1.3, 目的端口为 geneve 固定的 6081.

> 关于 UDP 隧道, 可以查看 https://switch-router.gitee.io/blog/udp-tunnel/

```
root@switch-router:/home/switch-router# ip -d addr show istioout
5: istioout: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UNKNOWN group default 
    link/ether c6:0c:46:c6:71:50 brd ff:ff:ff:ff:ff:ff promiscuity 0 minmtu 68 maxmtu 65485 
    geneve id 1001 remote 10.244.1.2 ttl auto dstport 6081 noudpcsum udp6zerocsumrx numtxqueues 1 numrxqueues 1 gso_max_size 65536 gso_max_segs 65535 
    inet 192.168.127.1/30 brd 192.168.127.3 scope global istioout
       valid_lft forever preferred_lft forever
```

其中封装后的报文目的地址变为 istioout 设备的 remote 地址 10.244.1.2.6081, 而源地址要根据路由后的结果决定.

根据【附录3】中的路由规则, 命中

```
10.244.1.2 dev veth08baa690 scope link 
```

报文被转发给 veth08baa690 (ztunnel Pod 在 host 侧的 veth)

```
14:49:20.418313 IP 10.244.1.1.2275 > 10.244.1.2.6081: Geneve, Flags [none], vni 0x3e9: IP 10.244.1.4.53696 > 10.244.2.5.8080: Flags [S], seq 3988947338, win 64240, options [mss 1460,sackOK,TS val 775716835 ecr 0,nop,wscale 7], length 0
```


##### 1st TCP 阶段4 阶段5 阶段6 SYN 报文进入 ztunnel Pod

报文顺着 veth 进入 ztunnel Pod, 进入协议栈, 查询 iptables【附录4】未命中, 查询路由【附录5】, 命中local路由

```
local 10.244.1.2 dev eth0 proto kernel scope host src 10.244.1.2
```
由于 UDP 隧道框架的特性, 该报文会送到 pistioout 设备解封装, 内层 SYN 报文以入接口为 pistioout 

查询iptables【附录4】，命中下面的规则

```
-A PREROUTING -i pistioin -p tcp -m tcp --dport 15008 -j TPROXY --on-port 15008 --on-ip 127.0.0.1 --tproxy-mark 0x400/0xfff
```

报文会被修改目的地址到 127.0.0.1.15008 上送本地, 原始的目的地址 10.244.2.5.8080 保存在 conntrack.

至此，client Pod 发送的 SYN 报文抵达 ztunnel. 

##### 1st TCP 阶段7 ztunnel 回复 SYNACK

ztunnel 组装的 SYNACK 为 10.244.2.5.8080 > 10.244.1.4.53696

查询路由【附录5】, 从 ztunnel 的 eth0 发出
```
10.244.1.0/24 via 10.244.1.1 dev eth0 src 10.244.1.2
```

##### 1st TCP 阶段8 阶段9 阶段10 SYNACK 报文到达 host

进入协议栈, 查询iptables【附录1】, 命中下面规则, 报文打上 0x210 标签, 
```
-A ztunnel-PREROUTING ! -s 10.244.1.2/32 -i veth08baa690 -j MARK --set-xmark 0x210/0x210
```
PS. 这条规则表示, 从ztunnel Pod出来的报文, 只要源IP不是ztunnel Pod IP本身, 就打上标签.

查询路由【附录3】, 命中下面这条, 表示报文会转发到 veth7a81c12c (与 client Pod 相连的 veth)
```
10.244.1.4 dev veth7a81c12c scope host 
```

路由查询完成后, 再查询iptables【附录1】, 命中下面这条, 这样依赖, 这条流就会打上 0x210 标签
```
-A ztunnel-FORWARD -m mark --mark 0x210/0x210 -j CONNMARK --save-mark --nfmask 0x210 --ctmask 0x210
```

SYN ACK 报文最终从 veth7a81c12c 回到 client Pod

##### 1st TCP 后续报文

后续 client Pod 发送的报文, 会iptables【附录1】, 打上 0x40 标签
```
-A ztunnel-PREROUTING ! -i veth08baa690 -m connmark --mark 0x210/0x210 -j MARK --set-xmark 0x40/0x40
```
查询路由【附录3】, 命中 table 102, 会直接送到 veth08baa690 (与 ztunnel Pod 相连的 veth), **不会再由 istioout 封装**
```
default via 10.244.1.2 dev veth08baa690 onlink 
10.244.1.2 dev veth08baa690 scope link 
```

#### 2nd TCP 路径

<p align="center"><img src="/assets/img/istio-ambient-path/pic3.png"></p>

##### 2nd TCP 阶段1 SYN 报文从 ztunnel 发出

第二段 TCP 连接由 ztunnel 向对端发起, 源地址保持为 client Pod 的地址 10.244.1.4, 目的地址端口为 10.244.2.5.15008

查询【附录5】, SYN 报文将从 eth0 发出

##### 2nd TCP 阶段2 阶段3 SYN 报文到达 host, 准备从节点发出

```
14:49:20.426051 IP 10.244.1.4.49137 > 10.244.2.5.15008: Flags [S], seq 3352392907, win 64240, options [mss 1460,sackOK,TS val 775716843 ecr 0,nop,wscale 7], length 0
```
查询 iptables【附录1】, 命中下面这条, SYN 报文打上 0x210 标签

```
-A ztunnel-PREROUTING ! -s 10.244.1.2/32 -i veth08baa690 -j MARK --set-xmark 0x210/0x210
```

查询路由【附录3】, 命中下面这条路由, 报文即将从 client 节点的 eth0 发出

```
10.244.2.0/24 via 172.18.0.3 dev eth0
```
再查询 iptables 【附录1】, 命中下面这条, 报文打上 0x210 标记

```
-A ztunnel-FORWARD -m mark --mark 0x210/0x210 -j CONNMARK --save-mark --nfmask 0x210 --ctmask 0x210
```

##### 2nd TCP 阶段4 阶段5 UDP隧道封装 SYN 报文

SYN 报文查询路由【附录7】, 命中 table 100 的这一条, 报文转发到 istioin

```
10.244.2.5 via 192.168.126.2 dev istioin src 10.244.2.1 
```

与 istioout 一样, istioin 也是一个隧道封装设备,

```
root@switch-router:/home/switch-router# ip -d addr show istioin
5: istioin: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UNKNOWN group default 
    link/ether 5a:e4:df:5e:5a:06 brd ff:ff:ff:ff:ff:ff promiscuity 0 minmtu 68 maxmtu 65485 
    geneve id 1000 remote 10.244.2.3 ttl auto dstport 6081 noudpcsum udp6zerocsumrx numtxqueues 1 numrxqueues 1 gso_max_size 65536 gso_max_segs 65535 
    inet 192.168.126.1/30 brd 192.168.126.3 scope global istioin
       valid_lft forever preferred_lft forever
```

其中封装后的报文目的地址变为 istioout 设备的 remote 地址 10.244.2.3.6081, 而源地址要根据路由后的结果决定.

根据【附录7】中的路由规则, 命中

```
10.244.2.3 dev veth07706a33 scope link 
```
报文被转发给 veth07706a33 (ztunnel Pod 在 host 侧的 veth)

##### 2nd TCP 阶段6 阶段7 阶段8 SYN 报文进入 ztunnel Pod

```
14:49:20.426191 IP 10.244.2.1.683 > 10.244.2.3.6081: Geneve, Flags [none], vni 0x3e8: IP 10.244.1.4.49137 > 10.244.2.5.15008: Flags [S], seq 3352392907, win 64240, options [mss 1460,sackOK,TS val 775716843 ecr 0,nop,wscale 7], length 0
```

查询iptables【附录8】未命中, 查询路由【附录9】命中 local 表, 上送本地
```
local 10.244.2.3 dev eth0 proto kernel scope host src 10.244.2.3 
```

由于 UDP 隧道框架的特性, 该报文会送到 pistioin 设备解封装, 内层 SYN 报文以入接口为 pistioin

查询iptables【附录8】，命中下面的规则

```
-A PREROUTING -i pistioin -p tcp -m tcp --dport 15008 -j TPROXY --on-port 15008 --on-ip 127.0.0.1 --tproxy-mark 0x400/0xfff
```

报文会被修改目的地址到 127.0.0.1.15008 上送本地, 原始的目的地址 10.244.2.5.15008 保存在 conntrack.

至此，client ztunnel Pod 发送的 SYN 报文抵达 server 侧 ztunnel. 

##### 2nd TCP 阶段9 ztunnel 回复 SYNACK

ztunnel 回复 SYNACK, 目的地址为 10.244.1.4.49137

查询路由【附录9】, 命中下面规则, 报文从 eth0 发送到 host
```
default via 10.244.2.1 dev eth0 
```

##### 2nd TCP 阶段10 SYN ACK 到达 host
```
14:49:20.426231 IP 10.244.2.5.15008 > 10.244.1.4.49137: Flags [S.], seq 1822713999, ack 3352392908, win 65160, options [mss 1460,sackOK,TS val 1932435212 ecr 775716843,nop,wscale 7], length 0
```
查询 iptables【附录6】, 报文打上 0x210 标记

```
-A ztunnel-PREROUTING ! -s 10.244.2.3/32 -i veth07706a33 -j MARK --set-xmark 0x210/0x210
```
查询 路由【附录7】, 命中下面这条, 报文将转发到 eth0

```
10.244.1.0/24 via 172.18.0.2 dev eth0
```

再查询 iptables【附录6】, 为该流打上 0x210 标记 (后续)
```
-A ztunnel-FORWARD -m mark --mark 0x210/0x210 -j CONNMARK --save-mark --nfmask 0x210 --ctmask 0x210
```

##### 2nd  TCP 阶段12 阶段13 阶段14 SYNACK 报文回到 client node

本文略

#### 3rd TCP 路径

第3段 TCP 连接的数据路径如图所示

<p align="center"><img src="/assets/img/istio-ambient-path/pic4.png"></p>

##### 3rd TCP 阶段1 阶段2 ztunnel 发起对 server Pod 的连接

源地址是  10.244.1.4, 目的地址是 10.244.2.5.8080

查询路由【附录9】命中下面这条, 从 eth0 发松到 host

```
10.244.2.0/24 via 10.244.2.1 dev eth0 src 10.244.2.3 
```

##### 3rd TCP 阶段3  转发 SYN 报文到 server Pod

```
14:49:20.428207 IP 10.244.1.4.42493 > 10.244.2.5.8080: Flags [S], seq 702419519, win 64240, options [mss 1460,sackOK,TS val 775716845 ecr 0,nop,wscale 7], length 0
```

查询 iptables【附录6】, 命中下面这条, 为报文打上 0x210 标记
```
-A ztunnel-PREROUTING ! -s 10.244.2.3/32 -i veth07706a33 -j MARK --set-xmark 0x210/0x210
```


查询 路由【附录7】, 命中下面这条, 报文转发到 veth935904a0 (与 Server Pod 相连的 veth)

```
10.244.2.5 dev veth935904a0 scope host 
```

随后查询 iptables【附录6】, 将流打上标记 0x210
```
-A ztunnel-FORWARD -m mark --mark 0x210/0x210 -j CONNMARK --save-mark --nfmask 0x210 --ctmask 0x210
```

##### 3rd TCP 阶段4 SYN 报文到达 server Pod

SYN 报文最终到达 server Pod

##### 3rd TCP 阶段5-8 SYN ACK 原路返回

只需注意一点, SYN ACK 报文从 Server Pod 到达 host 后, 由于流上有 0x210 标记. 因此会匹配到下面这条, 报文打上 0x40 标记

```
-A ztunnel-PREROUTING ! -i veth07706a33 -m connmark --mark 0x210/0x210 -j MARK --set-xmark 0x40/0x40
```
然后查询路由【附录7】会查询 table 102, 直达 ztunnel 对应的 veth

### 总结

- ambient 模式还是采取的 3段式的 TCP 连接方式，这一点与 sidecar 模式一样. 
- istioout 和 istioin 分别承担了 outbound 和 inbound 方向的 SYN 报文, 后续报文不会经过这两个隧道口 

### 附录

#### 附录1. client node iptables

```
-A ztunnel-PREROUTING -i istioin -j MARK --set-xmark 0x200/0x200
-A ztunnel-PREROUTING -i istioin -j RETURN
-A ztunnel-PREROUTING -i istioout -j MARK --set-xmark 0x200/0x200
-A ztunnel-PREROUTING -i istioout -j RETURN
-A ztunnel-PREROUTING -p udp -m udp --dport 6081 -j RETURN
-A ztunnel-PREROUTING -m connmark --mark 0x220/0x220 -j MARK --set-xmark 0x200/0x200
-A ztunnel-PREROUTING -m mark --mark 0x200/0x200 -j RETURN
-A ztunnel-PREROUTING ! -i veth08baa690 -m connmark --mark 0x210/0x210 -j MARK --set-xmark 0x40/0x40
-A ztunnel-PREROUTING -m mark --mark 0x40/0x40 -j RETURN
-A ztunnel-PREROUTING ! -s 10.244.1.2/32 -i veth08baa690 -j MARK --set-xmark 0x210/0x210
-A ztunnel-PREROUTING -m mark --mark 0x200/0x200 -j RETURN
-A ztunnel-PREROUTING -i veth08baa690 -j MARK --set-xmark 0x220/0x220
-A ztunnel-PREROUTING -p udp -j MARK --set-xmark 0x220/0x220
-A ztunnel-PREROUTING -m mark --mark 0x200/0x200 -j RETURN
-A ztunnel-PREROUTING -p tcp -m set --match-set ztunnel-pods-ips src -j MARK --set-xmark 0x100/0x100
-A PREROUTING -j ztunnel-PREROUTING
-A POSTROUTING -j ztunnel-POSTROUTING
-A ztunnel-OUTPUT -s 10.244.1.1/32 -p tcp -m owner --socket-exists -m set --match-set ztunnel-pods-ips dst -j MARK --set-xmark 0x220/0xffffffff
-A OUTPUT -j ztunnel-OUTPUT
-A ztunnel-INPUT -m mark --mark 0x220/0x220 -j CONNMARK --save-mark --nfmask 0x220 --ctmask 0x220
-A ztunnel-INPUT -m mark --mark 0x210/0x210 -j CONNMARK --save-mark --nfmask 0x210 --ctmask 0x210
-A INPUT -j ztunnel-INPUT
-A ztunnel-FORWARD -m mark --mark 0x220/0x220 -j CONNMARK --save-mark --nfmask 0x220 --ctmask 0x220
-A ztunnel-FORWARD -m mark --mark 0x210/0x210 -j CONNMARK --save-mark --nfmask 0x210 --ctmask 0x210
-A FORWARD -j ztunnel-FORWARD
```

#### 附录2. client node ipset

```
root@switch-router:/home/switch-router# ipset list
Name: ztunnel-pods-ips
Type: hash:ip
Revision: 0
Header: family inet hashsize 1024 maxelem 65536
Size in memory: 296
References: 2
Number of entries: 1
Members:
10.244.1.4
```

#### 附录3. client node ip rule && ip route

```
root@switch-router:/home/switch-router# ip rule
0:    from all lookup local 
100:    from all fwmark 0x200/0x200 goto 32766
101:    from all fwmark 0x100/0x100 lookup 101 
102:    from all fwmark 0x40/0x40 lookup 102 
103:    from all lookup 100 
32766:    from all lookup main 
32767:    from all lookup default 

root@switch-router:/home/switch-router# ip route show table local
local 10.244.1.1 dev veth08baa690 proto kernel scope host src 10.244.1.1 
local 10.244.1.1 dev veth7a81c12c proto kernel scope host src 10.244.1.1 
broadcast 127.0.0.0 dev lo proto kernel scope link src 127.0.0.1 
local 127.0.0.0/8 dev lo proto kernel scope host src 127.0.0.1 
local 127.0.0.1 dev lo proto kernel scope host src 127.0.0.1 
broadcast 127.255.255.255 dev lo proto kernel scope link src 127.0.0.1 
broadcast 172.18.0.0 dev eth0 proto kernel scope link src 172.18.0.2 
local 172.18.0.2 dev eth0 proto kernel scope host src 172.18.0.2 
broadcast 172.18.255.255 dev eth0 proto kernel scope link src 172.18.0.2 
broadcast 192.168.126.0 dev istioin proto kernel scope link src 192.168.126.1 
local 192.168.126.1 dev istioin proto kernel scope host src 192.168.126.1 
broadcast 192.168.126.3 dev istioin proto kernel scope link src 192.168.126.1 
broadcast 192.168.127.0 dev istioout proto kernel scope link src 192.168.127.1 
local 192.168.127.1 dev istioout proto kernel scope host src 192.168.127.1 
broadcast 192.168.127.3 dev istioout proto kernel scope link src 192.168.127.1 

root@switch-router:/home/switch-router# ip route show table 101
default via 192.168.127.2 dev istioout 
10.244.1.2 dev veth08baa690 scope link 

root@switch-router:/home/switch-router# ip route show table 102
default via 10.244.1.2 dev veth08baa690 onlink 
10.244.1.2 dev veth08baa690 scope link 

root@switch-router:/home/switch-router# ip route show table 100
10.244.1.2 dev veth08baa690 scope link 
10.244.1.4 via 192.168.126.2 dev istioin src 10.244.1.1 

root@switch-router:/home/switch-router# ip route show table main
default via 172.18.0.1 dev eth0 
10.244.0.0/24 via 172.18.0.4 dev eth0 
10.244.1.2 dev veth08baa690 scope host 
10.244.1.4 dev veth7a81c12c scope host 
10.244.2.0/24 via 172.18.0.3 dev eth0 
172.18.0.0/16 dev eth0 proto kernel scope link src 172.18.0.2 
192.168.126.0/30 dev istioin proto kernel scope link src 192.168.126.1 
192.168.127.0/30 dev istioout proto kernel scope link src 192.168.127.1 
```


#### 附录4 client ztunnel Pod iptables

```
-A PREROUTING -i pistioin -p tcp -m tcp --dport 15008 -j TPROXY --on-port 15008 --on-ip 127.0.0.1 --tproxy-mark 0x400/0xfff
-A PREROUTING -i pistioout -p tcp -j TPROXY --on-port 15001 --on-ip 127.0.0.1 --tproxy-mark 0x400/0xfff
-A PREROUTING -i pistioin -p tcp -j TPROXY --on-port 15006 --on-ip 127.0.0.1 --tproxy-mark 0x400/0xfff
-A PREROUTING ! -d 10.244.1.2/32 -i eth0 -p tcp -j MARK --set-xmark 0x4d3/0xfff
```

#### 附录5 client ztunnel Pod 路由

```
root@client-ztunnel:/# ip rule show
0:    from all lookup local 
20000:    from all fwmark 0x400/0xfff lookup 100 
20003:    from all fwmark 0x4d3/0xfff lookup 100 
32766:    from all lookup main 
32767:    from all lookup default 

root@client-ztunnel:/# ip route show table 100
local default dev lo scope host 

root@client-ztunnel:/# ip route show table main
default via 10.244.1.1 dev eth0 
10.244.1.0/24 via 10.244.1.1 dev eth0 src 10.244.1.2 
10.244.1.1 dev eth0 scope link src 10.244.1.2 
192.168.126.0/30 dev pistioin proto kernel scope link src 192.168.126.2 
192.168.127.0/30 dev pistioout proto kernel scope link src 192.168.127.2 

root@client-ztunnel:/# ip route show table local
broadcast 10.244.1.0 dev eth0 proto kernel scope link src 10.244.1.2 
local 10.244.1.2 dev eth0 proto kernel scope host src 10.244.1.2
broadcast 10.244.1.255 dev eth0 proto kernel scope link src 10.244.1.2 
broadcast 127.0.0.0 dev lo proto kernel scope link src 127.0.0.1 
local 127.0.0.0/8 dev lo proto kernel scope host src 127.0.0.1 
local 127.0.0.1 dev lo proto kernel scope host src 127.0.0.1 
broadcast 127.255.255.255 dev lo proto kernel scope link src 127.0.0.1 
broadcast 192.168.126.0 dev pistioin proto kernel scope link src 192.168.126.2 
local 192.168.126.2 dev pistioin proto kernel scope host src 192.168.126.2 
broadcast 192.168.126.3 dev pistioin proto kernel scope link src 192.168.126.2 
broadcast 192.168.127.0 dev pistioout proto kernel scope link src 192.168.127.2 
local 192.168.127.2 dev pistioout proto kernel scope host src 192.168.127.2 
broadcast 192.168.127.3 dev pistioout proto kernel scope link src 192.168.127.2 
```
#### 附录6 server node iptables

```
-A ztunnel-PREROUTING -i istioin -j MARK --set-xmark 0x200/0x200
-A ztunnel-PREROUTING -i istioin -j RETURN
-A ztunnel-PREROUTING -i istioout -j MARK --set-xmark 0x200/0x200
-A ztunnel-PREROUTING -i istioout -j RETURN
-A ztunnel-PREROUTING -p udp -m udp --dport 6081 -j RETURN
-A ztunnel-PREROUTING -m connmark --mark 0x220/0x220 -j MARK --set-xmark 0x200/0x200
-A ztunnel-PREROUTING -m mark --mark 0x200/0x200 -j RETURN
-A ztunnel-PREROUTING ! -i veth07706a33 -m connmark --mark 0x210/0x210 -j MARK --set-xmark 0x40/0x40
-A ztunnel-PREROUTING -m mark --mark 0x40/0x40 -j RETURN
-A ztunnel-PREROUTING ! -s 10.244.2.3/32 -i veth07706a33 -j MARK --set-xmark 0x210/0x210
-A ztunnel-PREROUTING -m mark --mark 0x200/0x200 -j RETURN
-A ztunnel-PREROUTING -i veth07706a33 -j MARK --set-xmark 0x220/0x220
-A ztunnel-PREROUTING -p udp -j MARK --set-xmark 0x220/0x220
-A ztunnel-PREROUTING -m mark --mark 0x200/0x200 -j RETURN
-A ztunnel-PREROUTING -p tcp -m set --match-set ztunnel-pods-ips src -j MARK --set-xmark 0x100/0x100
-A PREROUTING -j ztunnel-PREROUTING
-A POSTROUTING -j ztunnel-POSTROUTING
-A ztunnel-OUTPUT -s 10.244.2.1/32 -p tcp -m owner --socket-exists -m set --match-set ztunnel-pods-ips dst -j MARK --set-xmark 0x220/0xffffffff
-A OUTPUT -j ztunnel-OUTPUT
-A ztunnel-INPUT -m mark --mark 0x220/0x220 -j CONNMARK --save-mark --nfmask 0x220 --ctmask 0x220
-A ztunnel-INPUT -m mark --mark 0x210/0x210 -j CONNMARK --save-mark --nfmask 0x210 --ctmask 0x210
-A INPUT -j ztunnel-INPUT
-A ztunnel-FORWARD -m mark --mark 0x220/0x220 -j CONNMARK --save-mark --nfmask 0x220 --ctmask 0x220
-A ztunnel-FORWARD -m mark --mark 0x210/0x210 -j CONNMARK --save-mark --nfmask 0x210 --ctmask 0x210
-A FORWARD -j ztunnel-FORWARD
```

#### 附录7 server node ip rule && ip route
```
root@switch-router:/home/switch-router# ip rule show
0:    from all lookup local 
100:    from all fwmark 0x200/0x200 goto 32766
101:    from all fwmark 0x100/0x100 lookup 101 
102:    from all fwmark 0x40/0x40 lookup 102 
103:    from all lookup 100
32766:    from all lookup main 
32767:    from all lookup default

root@switch-router:/home/switch-router# ip route show table 100
10.244.2.3 dev veth07706a33 scope link 
10.244.2.5 via 192.168.126.2 dev istioin src 10.244.2.1 

root@switch-router:/home/switch-router# ip route show table 101
default via 192.168.127.2 dev istioout 
10.244.2.3 dev veth07706a33 scope link 

root@switch-router:/home/switch-router# ip route show table 102
default via 10.244.2.3 dev veth07706a33 onlink 
10.244.2.3 dev veth07706a33 scope link 

root@switch-router:/home/switch-router# ip route show table main
default via 172.18.0.1 dev eth0 
10.244.0.0/24 via 172.18.0.4 dev eth0 
10.244.1.0/24 via 172.18.0.2 dev eth0 
10.244.2.2 dev veth5bf1b956 scope host 
10.244.2.3 dev veth07706a33 scope host 
10.244.2.5 dev veth935904a0 scope host 
172.18.0.0/16 dev eth0 proto kernel scope link src 172.18.0.3 
192.168.126.0/30 dev istioin proto kernel scope link src 192.168.126.1 
192.168.127.0/30 dev istioout proto kernel scope link src 192.168.127.1 
```

#### 附录8 server ztunnel iptables

```
-A PREROUTING -i pistioin -p tcp -m tcp --dport 15008 -j TPROXY --on-port 15008 --on-ip 127.0.0.1 --tproxy-mark 0x400/0xfff
-A PREROUTING -i pistioout -p tcp -j TPROXY --on-port 15001 --on-ip 127.0.0.1 --tproxy-mark 0x400/0xfff
-A PREROUTING -i pistioin -p tcp -j TPROXY --on-port 15006 --on-ip 127.0.0.1 --tproxy-mark 0x400/0xfff
-A PREROUTING ! -d 10.244.2.3/32 -i eth0 -p tcp -j MARK --set-xmark 0x4d3/0xfff
```

#### 附录9 server ztunnel 路由

```
root@client-ztunnel:/#  ip rule show
0:    from all lookup local 
20000:    from all fwmark 0x400/0xfff lookup 100 
20003:    from all fwmark 0x4d3/0xfff lookup 100 
32766:    from all lookup main 
32767:    from all lookup default 

root@client-ztunnel:/#  ip route show table 100
local default dev lo scope host 

root@client-ztunnel:/#  ip route table main
default via 10.244.2.1 dev eth0 
10.244.2.0/24 via 10.244.2.1 dev eth0 src 10.244.2.3 
10.244.2.1 dev eth0 scope link src 10.244.2.3 
192.168.126.0/30 dev pistioin proto kernel scope link src 192.168.126.2 
192.168.127.0/30 dev pistioout proto kernel scope link src 192.168.127.2 

root@client-ztunnel:/#  ip route show table local
broadcast 10.244.2.0 dev eth0 proto kernel scope link src 10.244.2.3 
local 10.244.2.3 dev eth0 proto kernel scope host src 10.244.2.3 
broadcast 10.244.2.255 dev eth0 proto kernel scope link src 10.244.2.3 
broadcast 127.0.0.0 dev lo proto kernel scope link src 127.0.0.1 
local 127.0.0.0/8 dev lo proto kernel scope host src 127.0.0.1 
local 127.0.0.1 dev lo proto kernel scope host src 127.0.0.1 
broadcast 127.255.255.255 dev lo proto kernel scope link src 127.0.0.1 
broadcast 192.168.126.0 dev pistioin proto kernel scope link src 192.168.126.2 
local 192.168.126.2 dev pistioin proto kernel scope host src 192.168.126.2 
broadcast 192.168.126.3 dev pistioin proto kernel scope link src 192.168.126.2 
broadcast 192.168.127.0 dev pistioout proto kernel scope link src 192.168.127.2 
local 192.168.127.2 dev pistioout proto kernel scope host src 192.168.127.2 
broadcast 192.168.127.3 dev pistioout proto kernel scope link src 192.168.127.2 

```



