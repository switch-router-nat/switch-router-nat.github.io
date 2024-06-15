---
layout    : post
title     : "理解内核源端口选择--TCP"
date      : 2022-07-06
lastupdate: 2022-07-06
categories: Network(Kernel)
---

<p align="center"><img src="/assets/img/public/tcp.png"></p>

当 TCP 客户端向外发起连接的时候, 程序通常只会设置 TCP 四元组中的 Dst IP 和 Dst Port, 而 Src IP 和 Src Port 则是由本端内核决定.

<p align="center"><img src="/assets/img/tcp-port-selection/pic1.png"></p>

其中 Src IP 由路由决定, 而 Src Port 则在 ephemeral range 中选择. 后者可以通过 sysctl 命令查询.

```
root@switch-router: sysctl net.ipv4.ip_local_port_range
net.ipv4.ip_local_port_range = 32768	60999
```

那么, 内核何时确定 Src Port ? 如何选择 ? 内核可以选择相同的端口吗 ?  本文重点回答这几个问题.

### 何时确定 Src Port

这里要分两种情况: (1) connect 前不使用 bind()  (2) connect 前使用 bind()

#### 大部分情况: 不使用 bind()

大部分情况下, 客户端都不会使用 bind(), 此时, 内核在用户调用 connect() 时进行源端口选择. 

路径大概是:

```c
inet_stream_connect
 |- __inet_stream_connect
   |- tcp_v4_connect
     |- inet_hash_connect
        |- __inet_hash_connect          ====>   进行源端口选择
```

#### 小部分情况: 使用 bind()

如果在 connect() 前使用了 bind(), 则之后的端口会在此时就决定

```c
inet_bind
 |-- __inet_bind
   |-- sk->sk_prot->get_port(sk, snum)
     |- inet_csk_get_port               ====>  进行源地址绑定
```

### 如何选择

这里要先展示内核是如何组织相关的数据结构的. 全局数据 `tcp_hashinfo` 是内核 tcp 所有 sock 的组织者. 

其中包含了三组 hash 桶: 

ehash 保存已经 TCP_ESTABLISHED 的 sk
<mark>bhash 保存本端 sk 的占用情况</mark>
lhash 保存本端处于 LISTEN 状态的 sk

```
struct inet_hashinfo {

	struct inet_ehash_bucket	*ehash;
	spinlock_t			*ehash_locks;
	unsigned int			ehash_mask;
	unsigned int			ehash_locks_mask;

	struct kmem_cache		*bind_bucket_cachep;
	struct inet_bind_hashbucket	*bhash;
	unsigned int			bhash_size;

	/* The 2nd listener table hashed by local port and address */
	unsigned int			lhash2_mask;
	struct inet_listen_hashbucket	*lhash2;
};
```

bhash 的组织形式如下图所示:

<p align="center"><img src="/assets/img/tcp-port-selection/pic2.png"></p>

端口经过 hash 函数计算，作为链表元素挂到对应的链上, 旗下还有一条链表，保存所有使用该端口的 sk.

挂到同一个端口的 sk 的源端口是相同的, 但<mark>四元组一定不是完全一样的</mark>

#### 不使用 bind()

内核在`__inet_hash_connect()`进行源端口选择. 选择范围是 `ip_local_port_range` , 

它将从这个区间的 start 位置开始搜索遍历, 检查该端口是否**可用** 

> start 由 dst IP 和 dst Port 计算而来. 这样做可以让连接到同一个目的地址端口的连接选择端口是相近的.

<p align="center"><img src="/assets/img/tcp-port-selection/pic3.png"></p>

**可用**的判断条件是**是否存在相同的四元组**, 这通过查询 ehash 完成.

#### 使用 bind()

此时, tcp 在 `inet_csk_get_port` 进行端口选择. 

根据`bind()`的端口, 又分为两种情况： 0 和 非0

##### bind()到0

这表示应用程序让内核自己选择一个端口, 注意此时应用程序还没有`connect()`, 也就还不确定 Dst IP 和 Dst Port.

```
inet_csk_get_port
 |-- inet_csk_find_open_port
```

此时的端口选择和不使用 bind() 时，有相似的地方，也有不同的地方, 相似的地方是也是从 `ip_local_port_range` 遍历搜索, 

不同之处在于测试是否可用时, 不会再去 ehash 中查找 (因为此时连接还没有建立), 而且此时是一旦二元组(源地址+源端口)一致就判决为冲突

举个例子: 

假设现在有一条连接: 1.1.1.1:12345 <-> 2.2.2.2:8080 

此时本端 socket 绑定 1.1.1.1:12345 就不行, 即使这个 socket 将来是为了向 3.3.3.3 发起连接也不行.

##### bind()到非0

这种情况比较简单, 不需要内核选择了, 内核只需要检查指定的端口是否冲突就可以, 判决条件与`bind()`到0一致.

### 内核可以选择相同的端口吗 ?

完全可以！只要不使用 `bind()`, 且它们连接目标地址端口不完全一样

做一个简单的实验: 修改 `ip_local_port_range` 的范围到一个特定值 60100 (如此一来, 内核一定选择此端口)

```
sysctl -w net.ipv4.ip_local_port_range="60100 60100"
```

在本地启动两个 tcp server (代码在文末), 分别监听 8200 和 8201 端口, 再启动两个 tcp client 分别连接.

```
root@switch-router:/home/root # ss -npt | grep 127.0.0.1
ESTAB    0     0          127.0.0.1:8200       127.0.0.1:60100 users:(("python3",pid=848846,fd=4))                        
ESTAB    0      0         127.0.0.1:60100      127.0.0.1:8201  users:(("python3",pid=849082,fd=3))                        
ESTAB    0      0         127.0.0.1:60100      127.0.0.1:8200  users:(("python3",pid=848980,fd=3))                        
ESTAB    0     0          127.0.0.1:8201       127.0.0.1:60100 users:(("python3",pid=848859,fd=4)) 
```

可以看到两个客户端都如预期选择了 60100 端口.

### 附录

服务端代码

```python
import sys
import time

def main():
    if len(sys.argv) != 3:
        print("Usage: python tcp_client.py <server_ip> <port>")
        return

    server_ip = sys.argv[1]
    port = int(sys.argv[2])

    with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as s:
        s.connect((server_ip, port))
        time.sleep(30)


if __name__ == "__main__":
    main()
```

客户端代码

```python
import socket
import sys

def main():
    if len(sys.argv) != 2:
        print("Usage: python tcp_server.py <port>")
        return

    port = int(sys.argv[1])

    # Create a socket object
    server_socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)

    # Bind to the specified port
    server_socket.bind(('localhost', port))

    # Listen for connections
    server_socket.listen(5)

    print("Server started, listening on port %d..." % port)

    while True:
        # Accept client connections
        client_socket, addr = server_socket.accept()
        print('Received connection from %s' % str(addr))
        # Handle client connection here, without closing the sub-connections

if __name__ == "__main__":
    main()
```