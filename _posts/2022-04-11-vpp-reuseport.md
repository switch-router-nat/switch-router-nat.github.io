---
layout    : post
title     : "VPP: 跨进程 resueport"
date      : 2022-04-11
lastupdate: 2022-04-11
categories: VPP
---

<p align="center"><img src="/assets/img/public/fdio.png"></p>

> 本系列文章仅为学习使用 VPP 过程中的一些自用笔记

VPP 的 reuseport 的机制和内核有较大区别, 主要表现为 **<mark>始终使能同一个进程不同线程之间的resueort, 不支持不同进程的reuseport</mark>**

#### 内核 reuseport 机制

内核的 reuseport 机制经过多次(演进)\[<https://switch-router.gitee.io/blog/reuseport>], 最终以 reuseport group 为基础实现:

内核将监听同一个地址端口(打开reuseport)的套接字归于到同一个group中。当收到 TCP SYN 报文时，从命中的 group 中选择一个套接字.

<p align="center"><img src="/assets/img/vpp-reuseport/pic1.png"></p>

#### VPP reuseport 机制

VPP原生的reuseport的架构图如下图所示

<p align="center"><img src="/assets/img/vpp-reuseport/pic2.png"></p>

*图中涉及的术语解释如下*

*   application

    在 VPP 的视角下，一个应用进程就是一个 application，不管应用有多少线程，在VPP的视角下，它都是一个 application。可以通过 vcl.conf 文件为 application 指定一个 namespace，不同的 namespace 使用不同的 FIB 表和 session\_table 表。

*   app\_worker

    在 VPP 的视角下的，一个应用进程的一个工作线程是一个 app\_worker (使用数据结构 app\_worker\_t 表示)。每个 app\_worker 都属于唯一的 application，一个 application 可以包含多个 app\_worker。

*   app\_listener

    VPP视角下的监听器 (数据结构 app\_listener\_t 表示)。**<mark>一个 app\_listener属于唯一的 application</mark>**，一个 application 可以包含多个 app\_listener。app\_listener是从 application 的 listeners pool 中生成的。

    每个 app\_listener有一个 bitmap，记录哪些 app\_worker 正在使用该 listener。举例，如果有 2 个 app\_worker 同时使用一个 app\_listener(监听同一个 \[address, port])，则 app\_listener在收到新的连接请求时，会 select 其中一个 app\_worker。

*   listen session

    VPP 中处于 LISTEN 状态的一个会话(使用数据结构 session\_t 表示)，类比于 linux 中的监听套接字。

    listen session 都是从 VPP 的 #0 线程的 session pool 中分配， 它与 app\_listener 一一对应，listen session 记录了对应 app\_listener 在所属 application 的 listeners pool 中的 index；反过来，app\_listener 也记录了对应的 listen session 在 #0 线程 session pool 中的 index；从这个角度上来看，listen session 和 app\_listener 是一一关联的的，即**一个 listen session 也是属于唯一的 application**。除此之外，VPP 中的 session 也有明确的字段保存其属于哪一个 app\_worker (也就隐含了属于某一个 application)

*   session table

    VPP 存储 session 的多张 hash 表的集合。每个session table 与 IPv4 有关的有 2 张表，1张存储 established 和 listen session 的信息，另1张存储 half-open session 的信息。 hash表 KEY 为五元组信息，VALUE为 session handle  (session handle 为session index 和 创建session 的 VPP thread index 的组合，对listen session，thread index 始终为 0)。

    **每个 namespace 的 session table 是独立的**。

*   tcp connection listener

    VPP 中 TCP 层的 listener 表示 (使用数据结构 tcp\_connection\_t 表示)。从全局 tcp\_main 的 listener pool 中创建。与 session 一一对应 (listen ession是其逻辑上的 parent)。listen session 中的 connection\_index 记录了其在 tcp\_main 的 listener pool 中的 index。tcp connection listener中也记录了 listen 的 index。由于listen session 属于唯一的 application，因此 **<mark>tcp connection listener 在当前也是属于唯一的 application</mark>**。

从上面的架构图中可以看出，每个应用的 listener 和 session 都是独立的，且如果两个应用是同属于一个namespace，则它们会共用一个session table。当收到 peer 端的 SYN 请求时，VPP按照 session\_table----->session----->listener ------->app\_worker 的顺序找到一个线程。

#### 改进方案

为了让VPP能支持跨进程 reuseport, 我们可以也使用 reuseport group 的思想, 改造 VPP 内部相关数据结构的关系:

*   新增概念 **<mark>reuseport group</mark>** ，在使用 reuseport 时，VPP将监听同一个地址端口的 listen sessions 加入到同一个 reuseport group。

*   不再将使能了reuseport 的 listen session 以 \<key=协议地址端口，value=\[0, listen session index]> 加入 hashtable，取而代之的是 \<key=协议地址端口，value=\[1, reuseport group index]>

*   在收到SYN请求时，以 key=协议地址端口 查找hashtable ，若结果的高 32位为 0 (listen session都属于thread 0)，则说明低32位是一个 session index；若高32位为 1，则说明低 32 位是一个 reuseport group index，即以这种方式进行区分。

*   reuseport以轮询方式在app之间进行负载分担请求，即若app1、app2、app3都监听同一个地址端口，同时app1有1个worker，app2有2个worker，app3有3个worker。当收到源源不断的请求时，预期的响应worker为 app1-worker1、app2-worker1、app3-worker1、app1-worker1、app2-worker2、app3-worker2、app1-worker1、app2-worker1、app3-worker3......

<p align="center"><img src="/assets/img/vpp-reuseport/pic3.png"></p>

(完)
