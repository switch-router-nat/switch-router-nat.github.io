---
layout    : post
title     : "Cisco思科网络插件Contiv (四) 网络策略实践"
date      : 2018-05-16
lastupdate: 2018-05-16
categories: Container
---
## 网络策略的作用
Contiv可以通过网络策略来限制容器之间的访问行为，以实现用户对安全性的方面的要求。比如，我可以限制容器仅对源IP在特定范围的其他容器开放特定的端口，而拒绝其他IP地址的容器的访问。

## 搭建过程
### 环境准备
参考[Cisco思科网络插件Contiv (一) 环境部署](https://yacanliu.gitee.io/blog/2018/05/12/contiv-install/)搭建环境，由于本文并不关注Contiv网络的跨主机特性，因此只在一台宿主机上启动master进程和plugin进程就够了。
### 创建 contiv 网络
```
root@node-1:/home/yc/workspace# netctl --netmaster http://172.16.112.128:9999 network create --subnet=10.100.100.1/24 contiv-net
Creating network default:contiv-net 
```
 创建网络 **contiv-net**，指定子网范围是**10.100.100.1/24**

### 创建 Policy
```
root@node-1:/home/yc/workspace# netctl --netmaster http://172.16.112.128:9999  policy create web-policy
Creating policy default:web-policy 
```
创建名为**web-policy**的**Policy**， **Policy**的名字在租户(**tenant**)的命名空间(**namespace**)是唯一的，这里没有指定**tenant**参数， 因此使用的是** default **租户

### 添加 rule
Policy创建后是空的，我们还需要为Policy添加规则(rule). 一个Policy可以包含多个rule，最终的行为也是这些rule的叠加。每条rule包含以下信息：
 - `match criteria`  即**匹配规则**，定义了rule的入口条件，满足匹配规则的报文将执行设定的行为`action`， 匹配条件包括`protocol`、`port`、`ip-address`
      `protocol`可选参数为**TCP**、**UDP**、**ICMP**
      `port`为需要允许或禁止的端口号
      `ip-address`为出方向流量的目的**IP**，或者入方向流量的源**IP**
 - `direction`  可选参数为`inbound`为`outbound`，分别代表入方向和出方向的规则
 - `action` 可选参数为`allow`或`deny`
 - `priority`指定该** rule **的优先级，默认是**1**，数值越大的**rule**优先级越高
 - `from-group`   设置**rule**生效的**EndPoint**组的名字(入方向)
 - `to-group`  设置**rule**生效的**EndPoint**组的名字(出方向)
 - `from-network` 设置**rule**生效的**network**的名字(入方向)
 - `to-network` 设置**rule**生效的**network**的名字(出方向)

举例来说，我们要为我们的**httpd**容器设定以下行为：
 - 容器属于 **groupA**，开放`tcp/80`端口
 - 只允许属于 **groupB** 的容器访问`tcp/80`端口

首先创建 **groupA**和 **groupB**
```
root@node-1:/home/yc/workspace# netctl --netmaster http://172.16.112.128:9999  group create contiv-net groupA -policy=web-policy
Creating EndpointGroup default:groupA
root@node-1:/home/yc/workspace# netctl --netmaster http://172.16.112.128:9999  group create contiv-net groupB -policy=web-policy
Creating EndpointGroup default:groupB
```
创建 **rule 1**， 设置`deny`所有对`tcp/80`端口的访问
```
root@node-1:/home/yc/workspace# netctl --netmaster http://172.16.112.128:9999  policy rule-add web-policy 1 -direction=in -protocol=tcp -port=80 -action=deny 
```
创建 **rule 2**， 设置`allow` **groupB** 对`tcp/80`端口的访问
```
root@node-1:/home/yc/workspace# netctl --netmaster http://172.16.112.128:9999  policy rule-add web-policy 1 -direction=in -protocol=tcp -port=80 -action=deny 
```
查看设置的rule
```
root@node-1:/home/yc/workspace# netctl --netmaster http://172.16.112.128:9999  policy rule-ls web-policy
Incoming Rules:
Rule  Priority  From EndpointGroup  From Network  From IpAddress  To IpAddress  Protocol  Port  Action
----  --------  ------------------  ------------  ---------       ------------  --------  ----  ------
1     1                                                                         tcp       80    deny
2     10        groupB                                                          tcp       80    allow
Outgoing Rules:
Rule  Priority  To EndpointGroup  To Network  To IpAddress  Protocol  Port  Action
----  --------  ----------------  ----------  ---------     --------  ----  ------ 
```
### 启动 httpd 容器
```
root@node-1:/home/yc/workspace# docker run -itd --net=groupA --name=http-server httpd
4af399efa8646001599a3345231c5c34026139c5e2fd9012e1cdeff4b9dde71b  
```
启动名为**http-server**的容器，指定使用的网络为**groupA**，启动后通过docker network inspect命令可以看到容器分配的**IP**是**10.100.100.1**
```
root@node-1:/home/yc/workspace# docker network inspect groupA
[
    {
        "Name": "groupA",
        "Id": "20bcf90e8be6bb9c0dd4a5bcfdfa1812a91aab9bef137f60747b3ccc8602227a",
        "Created": "2018-09-30T09:12:58.292952948-07:00",
        "Scope": "global",
        "Driver": "netplugin",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "netplugin",
            "Options": {
                "group": "groupA",
                "network": "contiv-net",
                "tenant": "default"
            },
            "Config": [
                {
                    "Subnet": "10.100.100.0/24"
                }
            ]
        },
  "Internal": false,
        "Attachable": true,
        "Ingress": false,
        "ConfigFrom": {
            "Network": ""
        },
        "ConfigOnly": false,
        "Containers": {
            "4af399efa8646001599a3345231c5c34026139c5e2fd9012e1cdeff4b9dde71b": {
                "Name": "http-server",
                "EndpointID": "935a57e40dc4d508cc254e6abe0b6b9ee5ae0a7dca7ec843c9dff146c8c5d859",
                "MacAddress": "02:02:0a:64:64:01",
                "IPv4Address": "10.100.100.1/24",
                "IPv6Address": ""
            }
        },
        "Options": {
            "encap": "vxlan",
            "pkt-tag": "2",
            "tenant": "default"
        },
        "Labels": {}
    }
]  
```
### 启动 busybox 容器，访问httpd
启动两个busybox容器， 分别指定使用**groupA**和**groupB**网络, 尝试访问**httpd**容器的`tcp/80`端口

```
root@node-1:~# docker run -it --net=groupA --name=bbox-A busybox
/ # 
/ # wget http://10.100.100.1:80
Connecting to 10.100.100.1:80 (10.100.100.1:80)
wget: can't connect to remote host (10.100.100.1): Connection timed out
/ #  
```

```
root@node-1:~# docker run -it --net=groupB --name=bbox-B busybox
/ # 
/ # wget http://10.100.100.1:80
Connecting to 10.100.100.1:80 (10.100.100.1:80)
index.html           100% |***************************************************************************************************************************|    45  0:00:00 ETA
/ #  
```
可见， **bbox-B**可以访问**httpd**容器的`tcp/80`端口， 而**bbox-A**不能， 即**Policy**生效


