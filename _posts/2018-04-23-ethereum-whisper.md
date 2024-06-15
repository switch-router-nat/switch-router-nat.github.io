---
layout    : post
title     : "以太坊源码分析—Whisper"
date      : 2018-04-23
lastupdate: 2018-04-23
categories: Blockchain
---

## 前言
`Whisper`是以太坊中一项非常有趣的技术，它是一个基于身份的通信系统，被设计用于Dapp之间少量数据通信。`Whisper`协议运行在以太坊**p2p**协议框架之上，所有运行`Whisper`协议的节点（以下简称**节点**）组成一个`Whisper`网络。通过节点之间的消息转发，理论上，每个节点都可以收到所有`Whisper`消息。


## 特性
`Whisper`具有以下基本特性和概念

##### 通信加密
每一条`Whisper`消息在网络上都是加密传输的，可以选择非对称加密（椭圆曲线）和对称加密（AES GSM）两种加密算法之一。

##### Envelope(信封)
**Envelope**是网络中传输的`Whisper`消息的基本单位，它包含已加密的**原始消息**以及消息相关的控制信息：

 - Expiry time：消息的超时时刻，过了这个时刻，本消息不会被节点处理或者转发
 - TTL：消息的存活时间，一个消息在从被创建起，只能生存TTL的时间，过了这个时间之后，消息在网络中超时
 - Topic：消息的主题
 - AESNonce：采用AES对称密钥加密算法时使用的Nonce值
 - EnvNonce：用于PoW计算

当一个节点从一个**Peer**收到一个**Envelope**时，不管它自己管不关心里面的数据（Topic是否符合设置的值）， 它都会将这个**Envelope**转发给其他**Peer**，这是**Whisper**的固有机制。

##### Topic(主题)
每个**Envelope**上写明了自己封装消息的**Topic**，如果一个节点不关心这个**Topic**，那么它就不需要去试着打开（解密）这个**Envelope**。通常一个**Topic**对应一个消息加密时使用的Key（无论是对称还是非对称加密）。所以，如果一个节点收到了一个关心的**Topic**的**Envelope**时，它应该能打开这个**Envelope**

##### Filter(过滤器)
Dapp 可以在节点上安装多个**Filter**，每个**Filter**包含一组条件，只有满足这些条件的**Envelope**才能被打开，准确的说，不是节点打开**Envelope**，而是节点上安装的**Filter**打开**Envelope**，每个**Filter**有一个缓冲区可以存储解密后的消息，如果一个**Envelope**满足多个**Filter**，那么这个消息会存储在多个**Filter**中。**Filter**可以设置以下条件：

 - Topics：关心的主题的集合
 - Sender address：创建这个消息的节点
 - Recipient address：指定接收节点的地址
 - PoW requirement：消息需要的工作量证明
 - AcceptP2P：节点是否接收P2P消息，这类消息有特殊的用途

##### Proof of Work(工作量证明)
**Proof of Work**用来防止节点恶意大量发送消息，采用的算法和PoW共识算法差不多。消息的创建者需要找到一个nonce使得消息的Hash值小于一个值。这个值与消息的大小和TTL有关，消息越大，TTL越大，则找到nonce越困难，计算工作量的公式为


 ![formula][1]

其中$BestBit$为Hash值中从左往右第一个为**1**的bit所在的位置（这个值越大，则需要尝试nonce的次数越多）

## 源码分解

本部分主要涉及Whisper filter envelope ，它们的联系如下图：

#### Whisper 
**Whisper**表示一个协议实例，负责整个`Whisper`功能的运行，其中比较重要的字段如下：

 - protocol － Whisper协议的特定值，最重要的是其中的**Run**字段，它表示该协议的运行入口
 - filter - 本节点安装的所有**Filter**
 - privateKeys - 本节点存储的非对称密钥对的集合，一个Whisper实例可以保存多个密钥对
 - symKeys － 本届点存储的对称密钥的集合，一个Whisper实例可以保存多个对称密钥
 - envelopes － envelope池，保存所有待广播发送的信封。信封池的存储键值是envelope的Hash
 - expirations － 超时池，记录envelope的的过期时间，超时池的存储键值是unix时间，值是envelope的Hash
 - peers － 活跃的Peer节点的集合，数据来源是p2p底层
 - msgQueue  － 普通Whisper消息的envelope处理通道

在[以太坊源码分析—p2p节点发现与协议运行](https://segmentfault.com/a/1190000017000468)提到过，两个节点在底层连接建立后，会运行共同支持的协议的**Run**函数，对于Whisper协议来说，就是**HandlerPeer**函数。
![whisper](https://img-blog.csdn.net/20180702004829937?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2NoZW5tbzE4N0ozWDE=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)
**HandlePeer**最终运行在两个Go routine中，一个是**Whisper.runMessageLoop()**,它负责从底层读取消息，另一个是**Peer.update()**，它负责周期性的将**envelope池**中的未发送的**envelope**发送到对端并将过期的**envelope**删除。

#### Envelope
**Envelope**表示一个Whisper消息，它有两个来源
 1. 出方向通过**NewEnvelope()**构造
 2. 入方向从Peer节点接收

其重要的字段有

 - Topic － **Envelope**中的数据的主题，节点的**Filter**可以过滤感兴趣的主题进行解密
 - EnvNonce － 消息的创建者在PoW中找到的nonce
 - pow － 消息具有的pow值

#### 消息发送的典型过程
以下是本节点广播发送一小段数据*payload*，封装到**Envelope**，再加入到**Envelope池**的过程，其中`wh`表示**Whisper**实例
![send](https://img-blog.csdn.net/20180702004917396?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2NoZW5tbzE4N0ozWDE=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)
 1. 首先构造发送参数，包括原始数据*payload*，主题Topic等
 2. 利用发送参数构造**SentMessage**
 3. 根据发送参数将**SentMessage**封装到新创建的**Envelope**，这一步包括签名(sign)\加密(encrypt)\计算nonce(Seal)
 4. 将**Envelope**加入**Envelope池**
 

#### 消息接收的典型过程
以下是典型的`Whisper`消息接收过程，其中`w`表示**Whisper**实例
![receive](https://img-blog.csdn.net/20180702004948713?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2NoZW5tbzE4N0ozWDE=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

 - **w.runMessageLoop()**从底层收到`Whisper`消息，并解码成**Envelope**,加入**Envelope**池，调用**postEvent()**向w.messageQueue写入这个事件
 - 另一方面，当`w`通过**Start**启动后，从w.messageQueue接收事件，开始Filter匹配
 - 用已安装的每个Filter去匹配这个**Envelope**，如果匹配上，就将**Envelope**打开(解密)，最终将其放入**Filter**的缓冲区中。

## demo

写了一个whipser的chat-room demo，托管在[github](https://github.com/qshchenmo/ethereum-demo/tree/master/whisper-chat-room)上，感兴趣可以瞧瞧


  [1]: /img/bVbju4O