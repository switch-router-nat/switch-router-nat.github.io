---
layout    : post
title     : "Libnetwork CNM框架与实现"
date      : 2018-05-01
lastupdate: 2018-05-01
categories: Container
---
##  简介

`Libnetwork`是从docker1.6开始，逐渐将docker项目中的网络部分抽离出来形成的Lib，作用是为其他应用程序(如docker engine)提供一套抽象的容器网络模型，该模型也被称为 Container Network Model ，简称 CNM 。本文将描述以下内容

 - CNM框架模
 - Libnetwork的实现原理  
 - plugin demo

##  CNM 框架

<p align="center"><img src="/assets/img/libnetwork-CNM/arch.jpg"></p>

CNM模型下的docker网络模型如上所示。它由 Sandbox, Endpoint , Network 三种组件组成。 注意，该模型只是规定了三种组件各自的作用，他们都有各自的具体实现方式。

`Sandbox`:  Sandbox 包含了一个Container的网络相关的配置，如网卡Interface，路由表等。 Sandbox 在Linux上的典型实现是`Network namespace`。在Linux系统上的docker环境中，Container，Network namespace， Sandbox  这三者是绑定在一起的。一个 Sandbox 可以包含多个 Endpoint ，这些 Endpoint  可以来自多个 Network 
`Endpoint`:  Sandbox 加入 Network 的方式是通过 Endpoint 完成的。 Endpoint 的典型实现方式是`veth pair`，每个 Endpoint  都是由某个 Network 创建，创建后，它就归属于该 Network ，同时， Endpoint 还可以加入(Join)一个 Sandbox ，加入后，相当于该 Sandbox 也加入了此 Network 。
`Network`: Network 的一种典型实现是`Linux bridge`。一个 Network 可以创建多个 Endpoint 。将这些 Endpoint 加入到 Sandbox ，即实现了多个 Sandbox 的互通。

总结起来：如果要想两个Container之间可以直接通信，那么最简单的办法就是由一个 Network 创建两个 Endpoint ，分别加入这两个Container对应的 Sandbox 。

注意: 不同Network之间默认的隔离性是docker通过设置iptables完成的，通过改变iptables的设置，可以使得两个 Network 互通。

##  Libnetwork 实现

####核心对象

LibNetwork是CNM框架的实现，`Network`、`Sandbox`、`Endpoint`三种接口描述了前面的三种组件，三种接口分别在 network.go sandbox.go endpoint.go 实现

<p align="center"><img src="/assets/img/libnetwork-CNM/relation.png"></p>

三个接口需要实现的方法的关系如上图所示。注意：三种接口都有其对应的实现(如 Network 接口的实现为 network )。

 - `NetworkController` 为 docker engine提供创建Network的API，比如我们在使用命令  docker network create  创建网络时，都是通过controller创建。
 除此之外`NetworkController`接口的实现结构`controller`还维护了一张`Driver`的注册表(Registry),它记录了所有支持的Driver
 
 - `Driver` 每个Network都有对应的底层 Driver ，这些 Driver 负责在主机上真正实现需要网络功能(如创建Veth设备)。
 
 Driver有两种类型：
 
 - 内置型(如 Bridge \ Host \ None ) 
 - 插件型

<p align="center"><img src="/assets/img/libnetwork-CNM/req-reply.png"></p>

无论是哪种类型，其工作方式都类似， docker engine 发送请求， Libnetwork 做一些框架性的动作，然后将请求传给 Driver 做一些特异性的动作。两者的差别在于，内置性的 Driver 是内置在 Libnetwork 内部，它们也就是docker原生支持的网络类型。而插件型的 Driver 是运行在Host上，通过 socket 通道与 Libnetwork 之间进行通信。

####创建Controller

在 *controller.go* 中的 New() 是创建controller的入口。

```golang
func New(cfgOptions ...config.Option) (NetworkContriller, error) {
     c := controller{
         id: stringid.GenerateRandomID();
         sandboxes: sandboxTable{},
         .....
     }
     drvRegistry, err := drvregistry.New(......)
     for _, i := range getInitizers(c.cfg.Daemon.Experimental){
         drvRegistry.AddDriver(i.ntype,i.fn, dcfg)
     }
	
	 c.drvRegistry = drvRegistry
}
```
可以看到，首先是创建`controller`，此外 它还创建了`drvRegistry`记录注册的Driver，并将  getInitizers()  的返回内置型的 Driver 添加到 drvRegistry 上。
```
func getInitializers(experimental bool) []initialize ｛
    return []initializer {
        {bridge.Init, "bridge"},
        {host.Init, "host"},
        {macvlan.Init, "macvlan"},
        {null.Init, "null"},
        {remote.Init, "remote"},
        {overlay.Init, "overlay"},
        ......
    }
｝
```
 getInitizers()  包含的内置型Driver如上图所示，其中的每一项的 Init() 方法会被调用

####创建网络

<p align="center"><img src="/assets/img/libnetwork-CNM/create.png"></p>

创建  network  的入口 NewNetwork() 同样在 controller.go 中，它首先创建一个通用的  network  结构，进而解析出 其 Driver  类型，然后调用特定  Driver  的  CreateNetwork()  方法。注意，如果是该网络的类型不在  drvRegistry  上，则  Libnetwork  会尝试从  Plugin  路径 (/var/run/docker/plugins) 寻找  plugin-name.sock  .之后，向该 socket 发送创建网络的  request 

以  bridge  为例，其最后会调用到  drivers/bridge/bridge.go  的  createNetwork()  方法，该方法中，使用 netlink 提供的接口建立了  bridge net device  

####容器加入

在使用 docker run 启动容器时可以通过指定 net 参数将容器连接到特定的网络。入口是 docker 项目中的 container_operations.go 的 connectToNetwork() 

<p align="center"><img src="/assets/img/libnetwork-CNM/join.png"></p>

图中左侧部分是属于 `docker engine` ，中间部分属于 `Libnetwork` ， 右侧部分属于 `Driver` 。可以它首先创建Endpoint，创建 Sandbox 和 Endpoint，然后将 Endpoint 加入该 Sandbox 。

##  Plugin Demo
    
  将  docker  原生  Bridge  网络的功能插件化，制作名为  myPlugin  的网络插件，效果和原生的  Bridge  一样.

[Code](https://github.com/qshchenmo/docker-demo)

```
root@sb:/home/yc# ls -l /run/docker/plugins/
total 0
srw-rw---- 1 root root 0 8月  26 14:41 myplugin.sock

root@sb:/home/yc# docker network create --driver myplugin mynet
460db416c6f90fb16c499c5fd56b5e984d6472a113f5d8ed3f8633174159aa53
root@sb:/home/yc# docker network ls
NETWORK ID          NAME                DRIVER              SCOPE
6f5763ae335d        bridge              bridge              local
71d355df68ec        host                host                local
460db416c6f9        mynet               myplugin            local
91360b8a5fe4        none                null                local
docker run -tid --net=mynet busybox
617a314c4f69f835ab1a4e5cf9ce211a55e4651be4fa47e4ebd849c34c192e9d

root@sb:/home/yc# docker network inspect mynet
[
    {
        "Name": "mynet",
        "Id": "460db416c6f90fb16c499c5fd56b5e984d6472a113f5d8ed3f8633174159aa53",
        "Created": "2018-08-26T14:42:51.998760172+08:00",
        "Scope": "local",
        "Driver": "myplugin",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": {},
            "Config": [
                {
                    "Subnet": "172.18.0.0/16",
                    "Gateway": "172.18.0.1"
                }
            ]
        },
        "Internal": false,
        "Attachable": false,
        "Ingress": false,
        "ConfigFrom": {
            "Network": ""
        },
        "ConfigOnly": false,
        "Containers": {
            "617a314c4f69f835ab1a4e5cf9ce211a55e4651be4fa47e4ebd849c34c192e9d": {
                "Name": "gallant_wozniak",
                "EndpointID": "90ce305bea6a2054ee953f6fcebd0d23c94058026ac594d686f4d731464d45d8",
                "MacAddress": "",
                "IPv4Address": "172.18.0.2/16",
                "IPv6Address": ""
            }
        },
        "Options": {},
        "Labels": {}
    }
]


```