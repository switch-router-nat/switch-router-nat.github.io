---
layout    : post
title     : "Cisco思科网络插件Contiv (三) Plugin "
date      : 2018-05-15
lastupdate: 2018-05-15
categories: Container
---
## Contiv网络结构

![Contiv网络结构](https://s2.ax1x.com/2019/06/12/VRZWvV.png)

上图为Contiv的网络模型，大体上可分为`Master`和`Host Agent`两个组件,其中`Plugin`运行在每台宿主机上,  主要负责1. 与Container Runtime交互实现插件逻辑. 2. 配置底层 open vswitch进程实现具体的网络功能.

## Contiv-Plugin组件
#### Plugin Logic
**Plugin Logic** 是与**Container Runtime**交互的核心逻辑, 以常用的 **docker** 为例,  该逻辑即是实现CNM框架下所规定的种种接口, 实现与**Libnetwork**的消息交互, 关于**CNM**和**Libnetwork**, 请查看[Libnetwork与CNM框架与实现](https://blog.csdn.net/chenmo187J3X1/article/details/81984086)

#### Distributed KV Store
同 **Master** 中的作用一样, 下文将以etcd表示该数据库

#### Linux Host Routing/Switching
待完成
#### ARP/DNS Responder
待完成
#### Service LB
待完成
#### Route Distribution 
待完成

## Contiv-Plugin源码分析
#### plugin daemon 初始化
**Plugin** 进程的入口在 **/netplugin/netd.go** ,  主要完成命令行参数的解析. 然后创建一个**Agent** 
![Contiv-1](https://s2.ax1x.com/2019/06/12/VRZRg0.png)

#### plugin agent 创建
Agent的创建入口在 /netplugin/agent/agent.go, 
![Contiv-2](https://s2.ax1x.com/2019/06/12/VRZ23q.pngcontiv-2.png)
 - Cluster 初始化
    创建一个名为 objdbClient 的 etcd client,  它的作用是处理cluster级别的消息, 比如一台宿主机上的Plugin进程启动后需要让其他宿主机上的Master进程和Plugin进程感知到自己的存在,那么就需要通过这个client向etcd写入自己运行的服务, 这个过程也称为Service注册, 同时反过来,Plugin进程也可以通过该client侦测到其他plugin的启动, 这个过程称为 Peer Discovery. 言而言之,cluster 初始化使得plugin进程成为整个系统的一部分.
 - Netplugin 初始化
    Netplugin的初始化主要包括**State driver**的初始化和**Network driver**的初始化.  
    **State driver**的初始化主要是从**etcd**中获取**Master**进程写入的转发模式(**Fwd Mode**)和私有子网(**PvtSubnet**)等信息并校验和**Plugin**进程启动时的命令行一致, 如果没有得到, 说明   **Master**进程还没有启动, 需要等待.
   **Network driver**的初始化, 实际上是底层**ovs**的驱动的初始化, **Plugin**进程需要等待**ovs**进程连接上来.
 -  Container runtime plugin 初始化
   这部分要根据插件模式(**k8s** 或者 **docker**) 进行插件逻辑的初始化,  **k8s**对应**CNI**模型的插件. **docker**对应**CNM**模型的插件
    以**docker**为例, 这部分将启动一个Http Server, 用以响应 docker 进程发送过来的各类消息, 比如CreateNetwork,  CreateEndpoint等等

#### 状态恢复
![Contiv-3](https://s2.ax1x.com/2019/06/12/VRZhuT.png)
Contiv是跨主机容器网络,  因此, 当某台宿主机上的**Plugin**进程启动后, 需要将系统中其他节点已经创建的Contiv网络和容器加入网络的情况同步到本地, 这个过程就是状态恢复. 除了基本的network和endpoint信息, 可以看到这一步还需要同步BGP\EGP\ServiceLB\SvcProvider这些信息.

#### Post 初始化
![Contiv-4](https://s2.ax1x.com/2019/06/12/VRZgCn.png)
Post初始化完成两项工作. 
 - 启动Cluster 初始化时创建的 **objdbClient**, 使其完成**Service**注册和并开始**Peer Discovery**. 
 - 启动一个**REST Http Server**, 开放**9090**端口, 用户可以通过这个端口查看Plugin的一些工作情况

#### 启动外部事件响应循环
![Contiv-5](https://s2.ax1x.com/2019/06/12/VRZ64s.png)
在前面, **Plugin**进程从**etcd**中同步当前整个系统中的状态作为初始状态. 那么, 当网络状态发生变化后,**Plugin**进程也应该及时响应. 所以**Plugin**最后会启动外部事件响应循环, 这里根据事件类型的不同,实际会启动若干个**Go routine**, 在这些**routine**里, **Plugin**进程监视**etcd**中相应Key的变换, 并作出响应.






