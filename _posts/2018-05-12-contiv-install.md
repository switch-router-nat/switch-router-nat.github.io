---
layout    : post
title     : "Cisco思科网络插件Contiv (一) 安装"
date      : 2018-05-12
lastupdate: 2018-05-12
categories: Container
---

## 什么是Contiv

 Contiv ([官网](http://contiv.github.io/))是一个用于跨虚拟机、裸机、公有云或私有云的异构容器部署的开源容器网络架构。作为业界最强大的容器网络架构，Contiv具有2层、3层、overlay和ACI模式，能够与思科基础设施进行本地集成，并使用丰富的网络和安全策略将应用意图与基础设施功能进行映射。

Contiv是跨主机容器网络架构，因此，本文将两台虚拟机作为宿主机，在其上运行容器，验证其连通性。

## Contiv 网络结构
![Contiv网络模型](http://contiv.github.io/assets/images/Contiv-HighLevel-Architecture-a619d55a.png)

上图为Contiv的网络模型，大体上可分为`Master`和`Host Agent`两个组件,其中`Master`负责管理所有网络资源 (IP地址分配\租户管理\策略管理等等), 而`Host Agent`主要负责实现容器插件逻辑以及与下层网络驱动(ovs)的沟通.`Master`可以在多个节点运行多个实例以实现HA. `Host Agent`运行在每个节点上, 运行相应的Container RunTime(如docker)的Plugin. 
所有的网络信息资源都通过分布式KV存储单元(如ETCD)在每个节点间分享. 
`Master`上还运行了一个REST Server供外部来进行网络控制(如创建删除网络 ), netctl 工具可以连接这个Server来达到控制的目的.

## 环境部署

`host1`:  **ubuntu 16.04**+**docker 18.06**+**etcd 3.2.4**+ **ovs 2.5.4** +  **contiv 1.2.0**(**netmaster**+**netplugin**+**netctl**)
`host2`:  **ubuntu 16.04**+**docker 18.06**+ **ovs 2.5.4**+**contiv 1.2.0**(**netplugin**+**netctl**)

**etcd** 的下载安装方法见 [Flannel 环境搭建](https://blog.csdn.net/chenmo187J3X1/article/details/82590150)
**ovs** 使用 apt-get install 即可安装，具体安装方法查询网络
**contiv** 在 [github](https://github.com/contiv/netplugin/release) 下载 `netplugin-1.2.0.tar.bz2` 二进制文件 ，解压可以看到多个二进制可执行文件，将其中`netmaster` `netplugin` `netctl` 放到系统路径 (比如 */usr/local/bin*) , 其中`netmaster` 和 `netplugin` 是前一小节`Master`和`Host Agent`的实现

`host1 `的外部网卡的 **IP** 地址是 172.16.112.128
`host1 `的外部网卡的 **IP** 地址是 172.16.112.133

##### host1 运行 etcd
```
root@node-1:~# systemctl start etcd
root@node-1:~# systemctl status etcd
● etcd.service - Etcd Server
   Loaded: loaded (/lib/systemd/system/etcd.service; disabled; vendor preset: enabled)
   Active: active (running) since Fri 2018-09-21 01:34:56 PDT; 2h 10min ago
     Docs: https://github.com/coreos/etcd
 Main PID: 37808 (etcd)
    Tasks: 8
   Memory: 7.6M
      CPU: 24.690s
   CGroup: /system.slice/etcd.service
           └─37808 /usr/local/bin/etcd
```

##### host1 启动  netmaster
```
root@node-1:~# netmaster --etcd-endpoints http://172.16.112.128:2379 --mode docker --netmode vlan --fwdmode bridge
INFO[0000] Using netmaster log level: info              
INFO[0000] Using netmaster syslog config: nil           
INFO[Sep 21 03:59:14.020675885] Using netmaster log format: text             
INFO[Sep 21 03:59:14.032729154] Using netmaster mode: docker                 
INFO[Sep 21 03:59:14.033005583] Using netmaster network mode: vlan           
INFO[Sep 21 03:59:14.033087927] Using netmaster forwarding mode: bridge      
INFO[Sep 21 03:59:14.033171366] Using netmaster state db endpoints: etcd: http://172.16.112.128:2379 
INFO[Sep 21 03:59:14.033304358] Using netmaster docker v2 plugin name: netplugin 
INFO[Sep 21 03:59:14.033397300] docker v2plugin (netplugin) updated to netplugin and ipam (netplugin) updated to netplugin 
INFO[Sep 21 03:59:14.033481243] Using netmaster external-address: 0.0.0.0:9999 
INFO[Sep 21 03:59:14.068173614] Using netmaster internal-address: 172.16.112.128:9999 
INFO[Sep 21 03:59:14.068345036] Using netmaster infra type: default          
INFO[Sep 21 03:59:14.098425337] RPC Server is listening on [::]:9001  
......
INFO[Sep 21 03:59:14.211196969] Creating default tenant                      
INFO[Sep 21 03:59:14.211403602] Received TenantCreate: &{Key:default DefaultNetwork: TenantName:default LinkSets:{AppProfiles:map[] EndpointGroups:map[] NetProfiles:map[] Networks:map[] Policies:map[] Servicelbs:map[] VolumeProfiles:map[] Volumes:map[]}} 
INFO[Sep 21 03:59:14.212270603] Restoring ProviderDb and ServiceDB cache     
INFO[Sep 21 03:59:14.214864274] Registering service key: /contiv.io/service/netmaster/172.16.112.128:9999, value: {ServiceName:netmaster Role:leader Version: TTL:10 HostAddr:172.16.112.128 Port:9999 Hostname:} 
INFO[Sep 21 03:59:14.215062578] Registering service key: /contiv.io/service/netmaster.rpc/172.16.112.128:9001, value: {ServiceName:netmaster.rpc Role:leader Version: TTL:10 HostAddr:172.16.112.128 Port:9001 Hostname:} 
INFO[Sep 21 03:59:14.215215731] Registered netmaster service with registry   
INFO[Sep 21 03:59:14.215577345] Stop refreshing key: /contiv.io/service/netmaster/172.16.112.128:9999 
INFO[Sep 21 03:59:14.216141640] Stop refreshing key: /contiv.io/service/netmaster.rpc/172.16.112.128:9001 
INFO[Sep 21 03:59:14.217693322] Netmaster listening on 0.0.0.0:9999          
INFO[Sep 21 03:59:14.218048101] Ignore creating API listener on "172.16.112.128:9999" because "0.0.0.0:9999" covers it  
```

##### host1 启动  netplugin
```
root@node-1:~# netplugin --etcd-endpoints http://172.16.112.128:2379 --mode docker --netmode vlan --fwdmode bridge
INFO[0000] Using netplugin log level: info              
INFO[0000] Using netplugin syslog config: nil           
INFO[Sep 21 04:14:25.928104731] Using netplugin log format: text             
INFO[Sep 21 04:14:25.928303507] Using netplugin mode: docker                 
INFO[Sep 21 04:14:25.928597302] Using netplugin network mode: vlan           
INFO[Sep 21 04:14:25.928684356] Using netplugin forwarding mode: bridge      
INFO[Sep 21 04:14:25.928784788] Using netplugin state db endpoints: etcd: http://172.16.112.128:2379 
INFO[Sep 21 04:14:25.928946543] Using netplugin host: node-1                 
INFO[Sep 21 04:14:25.929640584] Using netplugin control IP: 172.16.112.128   
INFO[Sep 21 04:14:25.930269612] Using netplugin VTEP IP: 172.16.112.128      
INFO[Sep 21 04:14:25.930463923] Using netplugin vlan uplinks: []             
INFO[Sep 21 04:14:25.930549976] Using netplugin vxlan port: 4789             
INFO[Sep 21 04:14:25.959240581] Got global forwarding mode: bridge      
INFO[Sep 21 04:14:25.959429869] Got global private subnet: 172.19.0.0/16     
INFO[Sep 21 04:14:25.959481773] Using forwarding mode: bridge                
INFO[Sep 21 04:14:25.959529880] Using host private subnet: 172.19.0.0/16     
INFO[Sep 21 04:14:25.964467700] Initializing ovsdriver                       
INFO[Sep 21 04:14:25.964713009] Received request to create new ovs switch bridge:contivVxlanBridge, localIP:172.16.112.128, fwdMode:bridge 
INFO[Sep 21 04:14:26.075378465] Creating new ofnet agent for contivVxlanBridge,vxlan,[0 0 0 0 0 0 0 0 0 0 255 255 172 16 112 128],9002,6633
INFO[Sep 21 04:14:26.076282704] RPC Server is listening on [::]:9002    
......
```
此时  在 */run/docker/plugins/netplugin.sock* 目录 能看到`netplugin`创建出的.sock文件
```
root@node-1:~# ls -l /run/docker/plugins/netplugin.sock 
srwxr-xr-x 1 root root 0 Sep 21 04:14 /run/docker/plugins/netplugin.sock 
```
##### host2 启动  netplugin
同 host1 的启动方式(略)

##### host1 上创建网络
```
root@node-1:~# netctl --netmaster http://172.16.112.128:9999 network create --subnet 10.103.1.0/24 test-net
Creating network default:test-net
root@node-1:~# netctl --netmaster http://172.16.112.128:9999 network ls
Tenant   Network   Nw Type  Encap type  Packet tag  Subnet         Gateway  IPv6Subnet  IPv6Gateway  Cfgd Tag 
------   -------   -------  ----------  ----------  -------        ------   ----------  -----------  ---------
default  test-net  data     vxlan       0           10.103.1.0/24                                    
root@node-1:~# docker network ls
NETWORK ID          NAME                DRIVER              SCOPE
e600bbb6ba21        bridge              bridge              local
47b4619271d1        docker_gwbridge     bridge              local
193acc695266        host                host                local
af251471aaa4        none                null                local
32a8f6609407        test-net            netplugin           global 
```
这里使用 `netctl` 创建私有网络 **test-net** (10.103.1.0/24网段),  由于没有指定租户(--**tenant**参数), 因此创建的网络属于 **default** 租户. 使用 `docker network ls` 命令也可以看到创建的 **test-net** , 并可以看到其使用的 **driver** 类型正是 **netplugin** ,且 **Scope** 是 **global** ,表明这是一个跨主机网络.

在 host2 上可以看到该网络同样被创建
```
root@node-2:~# netctl --netmaster http://172.16.112.128:9999 network ls
Tenant   Network   Nw Type  Encap type  Packet tag  Subnet         Gateway  IPv6Subnet  IPv6Gateway  Cfgd Tag 
------   -------   -------  ----------  ----------  -------        ------   ----------  -----------  ---------
default  test-net  data     vxlan       0           10.103.1.0/24                                    
root@node-2:~# netctl --netmaster http://172.16.112.128:9999 network ls
Tenant   Network   Nw Type  Encap type  Packet tag  Subnet         Gateway  IPv6Subnet  IPv6Gateway  Cfgd Tag 
------   -------   -------  ----------  ----------  -------        ------   ----------  -----------  ---------
default  test-net  data     vxlan       0           10.103.1.0/24                                    
root@node-2:~# docker network ls
NETWORK ID          NAME                DRIVER              SCOPE
c42850c2d8c4        bridge              bridge              local
d925b7303139        docker_gwbridge     bridge              local
390b696af601        host                host                local
d1016e4ad403        none                null                local
32a8f6609407        test-net            netplugin           global
```
##### host1 host2 上启动容器 
```
root@node-1:~# docker run --net test-net --name contiv-bbox -tid busybox
420722e232a2bf24700976b514e405915afef56697c15aa630f37b1683714d59 
```
**host1** 上启动 **busybox** 容器, 指定使用 **test-net** 网络
```
root@node-1:~# docker run --net test-net --name cbox1 -tid busybox
cbd3cf2ecfbae8f88e49f45db6a64c841cfb8f228e8c556af8d3919762f61a65
root@node-1:~# docker exec cbox1 ifconfig
eth0      Link encap:Ethernet  HWaddr 02:02:0A:67:01:03  
          inet addr:10.103.1.3  Bcast:10.103.1.255  Mask:255.255.255.0
          UP BROADCAST RUNNING MULTICAST  MTU:1450  Metric:1
          RX packets:21 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0 
          RX bytes:2619 (2.5 KiB)  TX bytes:0 (0.0 B) 
          ......
```
可以看到容器上 **eth0** 分配的 **IP** 地址 **10.103.1.3** 正是属于之前创建的Contiv网络

在 **host2** 上进行同样的操作, **eth0** 分配的 **IP** 地址 **10.103.1.4**
```
root@node-2:~# docker exec cbox2 ifconfig
eth0      Link encap:Ethernet  HWaddr 02:02:0A:67:01:04  
          inet addr:10.103.1.4  Bcast:10.103.1.255  Mask:255.255.255.0
          UP BROADCAST RUNNING MULTICAST  MTU:1450  Metric:1
          RX packets:21 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0 
          RX bytes:2619 (2.5 KiB)  TX bytes:0 (0.0 B) 
 ......
```
##### 容器的连通
从**host1** 上的容器**cbox1** ping **host2** 上的容器**cbox2**, 可见是可以通信的
```
root@node-1:~# docker exec cbox1 ping -c 4 cbox2
PING cbox2 (10.103.1.4): 56 data bytes
64 bytes from 10.103.1.4: seq=0 ttl=64 time=10.661 ms
64 bytes from 10.103.1.4: seq=1 ttl=64 time=0.628 ms
64 bytes from 10.103.1.4: seq=2 ttl=64 time=0.450 ms
64 bytes from 10.103.1.4: seq=3 ttl=64 time=0.604 ms

--- cbox2 ping statistics ---
4 packets transmitted, 4 packets received, 0% packet loss
round-trip min/avg/max = 0.450/3.085/10.661 ms 
```




