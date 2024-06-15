---
layout    : post
title     : "Cisco思科网络插件Contiv (二) Master"
date      : 2018-05-14 
lastupdate: 2018-05-14 
categories: Container
---
## Contiv网络结构


![Contiv网络结构](https://s2.ax1x.com/2019/06/12/VRZ0u8.png)
上图为Contiv的网络模型，大体上可分为`Master`和`Host Agent`两个组件,其中`Master`负责管理所有网络资源 (IP地址分配\租户管理\策略管理等等)

### Contiv-Master 组件
#### Distributed KV Store
**Distributed KV Store**, 即分布式键值存储, 它是跨主机容器网络的重要组成部分, 各个宿主机通过它进行配置数据和运行数据的同步, **Contiv**也不例外. **Contiv**提供**Etcd**和**Consul**两种实现. 无论是哪一种, 信息都是以**Key-Value Pair**的形式存储,并且Key都是 **/contiv.io/** 开头的形式. 比如 **/contiv.io/state/xxx** 记录配置信息, **/contiv.io/oper/xxx** 记录运行信息

#### HA
为了实现**HA**(高可用),  `Master` 通常在多台宿主机上运行运行多个进程实例, 但同一时刻, 有且只有一个宿主机上的进程以 **Leader** 角色运行, 其余都以 **Follow** 角色运行. Master进程启动后, 首先以**Follow** 角色运行, 并且尝试去获取分布式数据库(Distributed KV Store)中的一把 **Leader Lock** (路径为/contiv.io/lock/netmaster/leader), 若能获取到, 则将角色切换为 **Leader** , 若不能获取到,则还是以 **Follow** 角色运行.

#### REST Server
**Service**表示**Master**对外提供的服务，**Master**进程启动后, 会将自身运行的**Service**信息（**IP**地址 \ 端口号 \ 角色）写入分布式数据库，这个过程称为 **Service** 注册。在当前版本中**Master** 会注册 **netmaster** 和 **netmaster.rpc** 两个服务，前者开放**9999**端口供`netctl` 控制整个系统，后者开放**9001**端口供**OpenFlow**控制器使用。

#### Policy Engine
**Policy Engine**用来管理控制容器之间的网络流量隔离\优先级策略. 比如设置容器A禁止除了容器B以外的其他容器访问XXXX其端口. **Master** 的 **REST Server**在 **/api/v1/policys/{keys}** 和  **/api/v1/rules/{keys}** 都设置了相应的Handler, 当用户通过**netctl** 的命令添加或删除策略时, **Master**会调用对应的Handler, 最终通过设置 **iptable** 防火墙完成既定功能.

### Contiv-Master 源码分析
#### master daemon 初始化
**Master**进程的入口在 *netmaster/main.go*,  它主要进行命令行参数的解析, 将解析的结果放入**daemon.MasterDaemon** 结构, 之后调用 **MasterDaemon** 的 **Init()** 方法初始化进程
![Contiv-1](https://s2.ax1x.com/2019/06/12/VRZagP.png)

#### 运行netmaster状态机
![Contiv-2](https://s2.ax1x.com/2019/06/12/VRZdjf.png)
接着,  **Master**进程启动状态机, 它只有两个状态,即前面提到的**Master**进程的角色,  启动之初都是以 **Follower** 角色运行, 若是能获得 **Leader** 锁, 则调用**becomeLeader()**进入 **Leader** 状态

#### Leader 运行
![Contiv-3](https://s2.ax1x.com/2019/06/12/VRZU3t.png)
 - 以**Leader**角色运行时, **Master**进程首先创建一个**APIController**, 它创建**REST Server**提供给**netctl** , 设置各个URL对应的相应的处理函数.
 - **Service** 注册, 注册 **netmaster** 和 **netmaster.rpc** 服务
 - 初始化策略管理器, 这一步主要是从数据库中恢复各个**Policy**
 - 设置面向**plugin**的**Server**, 例如, 当**plugin**要向**Master**申请IP时,就会向"plugin/allocAddress"发送请求.