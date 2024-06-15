---
layout    : post
title     : "深入浅出TCP中的SYN-Cookies"
date      : 2021-05-25
lastupdate: 2021-05-25
categories: Network(Kernel)
---

<p align="center"><img src="/assets/img/tcp-syncookie/logo.png"></p>

### SYN Flood 攻击

`TCP`连接建立时，客户端通过发送`SYN`报文发起向处于监听状态的服务器发起连接，服务器为该连接分配一定的资源，并发送`SYN+ACK`报文。对服务器来说，此时该连接的状态称为`半连接`(`Half-Open`)，而当其之后收到客户端回复的`ACK`报文后，连接才算建立完成。在这个过程中，如果服务器一直没有收到`ACK`报文(比如在链路中丢失了)，服务器会在超时后重传`SYN+ACK`。

<p align="center"><img src="/assets/img/tcp-syncookie/syn-handshake.png"></p>

如果经过多次超时重传后，还没有收到, 那么服务器会回收资源并关闭`半连接`，仿佛之前最初的`SYN`报文从来没到过一样！

<p align="center"><img src="/assets/img/tcp-syncookie/attacker.png"></p>

这看上一切正常，但是如果有坏人**故意**大量不断发送伪造的`SYN`报文，那么服务器就会分配大量注定无用的资源，并且服务器能保存的半连接的数量是有限的！所以当服务器受到大量攻击报文时，它就不能再接收正常的连接了。换句话说，它的服务不再可用了！这就是`SYN Flood`攻击的原理，它是一种典型的`DDoS`攻击。

### 连接请求的关键信息

`Syn-Flood`攻击成立的关键在于服务器资源是有限的，而服务器收到请求会分配资源。通常来说，服务器用这些资源保存此次请求的关键信息，包括请求的来源和目(五元组)，以及`TCP`选项，如最大报文段长度`MSS`、时间戳`timestamp`、选择应答使能`Sack`、窗口缩放因子`Wscale`等等。当后续的`ACK`报文到达，三次握手完成，新的连接创建，这些信息可以会被复制到连接结构中，用来指导后续的报文收发。

那么现在的问题就是服务器如何在**不分配**资源的情况下

 1. 验证之后可能到达的`ACK`的有效性，保证这是一次完整的握手
 2. 获得`SYN`报文中携带的`TCP`选项信息

### SYN cookies 算法

`SYN Cookies`[算法](https://en.wikipedia.org/wiki/SYN_cookies)可以解决上面的第`1`个问题以及第`2`个问题的一部分

我们知道，`TCP`连接建立时，双方的起始报文序号是可以**任意**的。`SYN cookies`利用这一点，按照以下规则构造初始序列号：

 - 设`t`为一个缓慢增长的时间戳(典型实现是每64s递增一次)
 - 设`m`为客户端发送的`SYN`报文中的`MSS`选项值
 - 设`s`是连接的元组信息(源IP,目的IP,源端口，目的端口)和`t`经过密码学运算后的`Hash`值，即`s = hash(sip,dip,sport,dport,t)`，`s`的结果取低 **24** 位

则初始序列号`n`为：

 - 高 **5** 位为`t mod 32`
 - 接下来**3**位为`m`的编码值
 - 低 **24** 位为`s`

当客户端收到此`SYN+ACK`报文后，根据`TCP`标准，它会回复`ACK`报文，且报文中`ack = n + 1`，那么在服务器收到它时，将`ack - 1`就可以拿回当初发送的`SYN+ACK`报文中的序号了！服务器巧妙地通过这种方式间接保存了一部分`SYN`报文的信息。

接下来，服务器需要对`ack - 1`这个序号进行检查：

 - 将高 **5** 位表示的`t`与当前之间比较，看其到达地时间是否能接受。
 - 根据`t`和连接元组重新计算`s`，看是否和低 **24** 一致，若不一致，说明这个报文是被伪造的。
 - 解码序号中隐藏的`mss`信息

到此，连接就可以顺利建立了。

####  SYN Cookies 缺点

既然`SYN Cookies`可以减小资源分配环节，那为什么没有被纳入`TCP`标准呢？原因是`SYN Cookies`也是有代价的：

 1. `MSS`的编码只有**3**位，因此最多只能使用 **8** 种`MSS`值
 2. 服务器必须拒绝客户端`SYN`报文中的其他只在`SYN`和`SYN+ACK`中协商的选项，原因是服务器没有地方可以保存这些选项，比如`Wscale`和`SACK`
 3. 增加了密码学运算

#### Linux 中的 SYN Cookies

`Linux`上的`SYN Cookies`实现与`wiki`中描述的算法在序号生成上有一些区别，其`SYN+ACK`的序号通过下面的公式进行计算：

> 内核编译需要打开 **CONFIG_SYN_COOKIES** 

```
seq = hash(saddr, daddr, sport, dport, 0, 0) + req.th.seq + t << 24 + (hash(saddr, daddr, sport, dport, t, 1) + mss_ind) & 0x00FFFFFF
```
其中，`req.th.seq`表示客户端的`SYN`报文中的序号，`mss_ind`是客户端通告的`MSS`值得编码，它的取值在比较新的内核中有 **4** 种(老的内核有 **8** 种), 分别对应以下 **4** 种值

```
static __u16 const msstab[] = {
	536,
	1300,
	1440,	/* 1440, 1452: PPPoE */
	1460,
};
```

感兴趣的可以顺着以下轨迹浏览调用顺序

收到SYN报文：
```
tcp_v4_rcv
  |
  |- __inet_lookup_skb
  |- tcp_v4_do_rcv
      |
      |- tcp_rcv_state_process
          |
          |- tcp_conn_request
             |
             |- cookie_init_sequence
                |
                |- cookie_v4_init_sequence
                   |
                   |- __cookie_v4_init_sequence
                      |
                      |-- secure_tcp_syn_cookie
```

收到ACK报文
```
tcp_v4_rcv
 |
 |- __inet_lookup_skb
 |- tcp_v4_do_rcv
   |
   |- tcp_v4_cookie_check  //  SYN_RCV
      |
      |- cookie_v4_check
   |- tcp_child_process   
      |
      |- tcp_rcv_state_process
         |
         |- tcp_ack
```

#### SYN Cookies 与时间戳

如果服务器和客户端**都**打开了时间戳选项，那么服务器可以将客户端在`SYN`报文中携带了`TCP`选项的使能情况暂时保存在时间戳中。当前使用了低 **6** 位，分别保存`Wscale`、`SACK`和`ECN`。

<p align="center"><img src="/assets/img/tcp-syncookie/ts.png"></p>
客户端会在`ACK`的`TSecr`字段，把这些值带回来。

### 实验

> **Linux**中的`/proc/sys/net/ipv4/tcp_syncookies`是内核中的`SYN Cookies`开关,`0`表示关闭`SYN Cookies`；`1`表示在新连接压力比较大时启用`SYN Cookies`,`2`表示始终使用`SYN Cookies`。

本实验是在`4.4.0`内核运行的，服务端监听`50001`端口，`backlog`参数为`3`。同时，模拟不同的客户端注入`SYN`报文。

测试代码见本文末

#### 不开启 SYN Cookies

```
echo 0 > /proc/sys/net/ipv4/tcp_syncookies
```

可以看到，在收到`3`个`SYN`报文后，服务器不再响应新的连接请求了，这也就是`SYN-Flood`的攻击方式。
<p align="center"><img src="/assets/img/tcp-syncookie/pcap1.png"></p>
#### 有条件使用 SYN Cookies
```
echo 1 > /proc/sys/net/ipv4/tcp_syncookies
```
<p align="center"><img src="/assets/img/tcp-syncookie/pcap2.png"></p>
由于服务器的`backlog`参数为`3`，因此图中的从第`4`个`SYN+ACK`(**#8**报文)开始使用`SYN Cookies`。

从时间戳可以看出，**#8**报文(44167748)比 **#6**号报文(44167796)还要小。

```bash
44167748 = 0x2A1F244 ,最后低6位是 0b000100 ,与SYN报文中 wscale = 4 是相符的
```

### 小结

`SYN Cookie`技术可以让服务器在收到客户端的`SYN`报文时，不分配资源保存客户端信息，而是将这些信息保存在`SYN+ACK`的初始序号和时间戳中。对正常的连接，这些信息会随着`ACK`报文被带回来。

### REF

- [SYN Flood Attack](https://www.cloudflare.com/learning/ddos/syn-flood-ddos-attack/)
- [Improving syncookies](https://lwn.net/Articles/277146/)

### Appendix

测试代码
```c
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <netinet/ip.h>
#include <netinet/tcp.h>
#include <net/if.h>
#include <sys/ioctl.h>
#include <linux/if_tun.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <pthread.h>

#define PCKT_LEN 1024
#define BACKLOG 3
#define TUN_ADDR "192.168.2.1"
#define SPOOF_NET "192.168.3.0"
#define SPOOF_PREFIX "192.168.3."

#define COUNT 8

const char* spoof_ip_list[COUNT] = {"192.168.3.1",
         "192.168.3.2",
         "192.168.3.3",
         "192.168.3.4",
         "192.168.3.5",
         "192.168.3.6",
         "192.168.3.7",
         "192.168.3.8"}; 

const uint16_t spoof_mss[COUNT] = {536, 1300, 1440, 1460, 536, 1300, 1440, 1460};

const uint32_t spoof_tsp[COUNT] = {1000, 2000, 3000, 4000, 5000,6000, 7000, 8000};

const uint8_t spoof_wscale[COUNT] = {1, 2, 3, 4, 1, 2, 3, 4};

#define TUN_PORT 50001

struct psdhdr{
 uint32_t saddr;
 uint32_t daddr;
 char zero;
 char protocol;
 uint16_t tcplen;
};

struct mss_opt{
 uint8_t kind; // = 2
 uint8_t length; // = 4
 uint16_t mss;  
}__attribute__((packed));

struct tstamp_opt{
 uint8_t kind; // = 8 
 uint8_t length; // = 10
 uint32_t tsval;  
 uint32_t tsecr;
 uint8_t nop[2];
}__attribute__((packed));

struct wscale_opt{
 uint8_t kind; // = 3 
 uint8_t length; // = 3
 uint8_t scale;  
 uint8_t nop;
}__attribute__((packed));

uint16_t calc_cksm(void *pkt, int len)
{
    uint16_t *buf = (uint16_t*)pkt;
    uint32_t cksm = 0;
    while(len > 1)
    {
        cksm += *buf++;
        cksm = (cksm >> 16) + (cksm & 0xffff);
        len -= 2;
    }
    if(len)
    {
        cksm += *((uint8_t*)buf);
        cksm = (cksm >> 16) + (cksm & 0xffff);
    }
    return (uint16_t)((~cksm) & 0xffff);
} 


unsigned short tcp_checksum (struct iphdr *ip, struct tcphdr* th, char* opt, int optlen)
{
	uint16_t sum = 0;
	char buf[PCKT_LEN];
	int chksumlen = 0;
	struct psdhdr psdhdr;

	memset(buf, 0, PCKT_LEN);

	psdhdr.saddr = ip->saddr;
	psdhdr.daddr = ip->daddr;
	psdhdr.zero = 0;
	psdhdr.protocol = ip->protocol;
	psdhdr.tcplen = htons(sizeof(struct tcphdr) + optlen);

	memcpy(&buf[0], &psdhdr, sizeof(struct psdhdr));

	chksumlen += sizeof(struct psdhdr);

	memcpy(&buf[chksumlen], th, sizeof(struct tcphdr));

	chksumlen += sizeof(struct tcphdr);

	if (optlen > 0)
	{
		memcpy(&buf[chksumlen], opt, optlen);
		chksumlen += optlen;
	}

	sum = calc_cksm(buf, chksumlen);

	return sum; 
}


int tun_create(int flags)
{
    int fd, err;
    struct ifreq ifr;

    if ((fd = open("/dev/net/tun", O_RDWR)) < 0){
        return fd;
    }

    memset(&ifr, 0, sizeof(ifr));
    ifr.ifr_flags = flags;

    if ((err = ioctl(fd, TUNSETIFF, (void*)&ifr)) < 0 )
    {
        close(fd);
        return err;
    }

    if (strcmp(ifr.ifr_name, "tun0")) {
        close(fd);
        return -1;
    }

    return fd;
} 

int tun_setup(char* tundev)
{
    struct ifreq ifr;
    int sockfd;
    int err;
    
    memset(&ifr, 0, sizeof(ifr));
    snprintf(ifr.ifr_name, (sizeof(ifr.ifr_name) - 1), "%s", tundev);
    
    sockfd = socket(AF_INET, SOCK_DGRAM, 0);
    if (sockfd < 0)
    {
        return err;
    }
        
    if((err = ioctl(sockfd, SIOCGIFFLAGS, (void *)&ifr)) < 0 ) 
    {
        return err;
    }

    ifr.ifr_flags |= IFF_UP;
    if((err = ioctl(sockfd, SIOCSIFFLAGS, (void *)&ifr)) < 0 ) 
    {
        return err;
    }

    close(sockfd);

    return 0;
} 

/* Configure a local IPv4 address and netmask for the device */
int tun_set_address(const char* dev,
                     const char* ip,
                      int prefix_len)
{
    char command[128];

    memset(command, 0, sizeof(command));
    
    sprintf(command, "ip addr add %s/%d dev %s > /dev/null 2>&1", ip, prefix_len, dev);

    int result = system(command);
    
    return result;
} 

int tun_set_route()
{
    char command[128];

    memset(command, 0, sizeof(command));

    sprintf(command,
           "ip route add %s/24 via %s > /dev/null 2>&1", // ip -4 route add 
            SPOOF_NET, TUN_ADDR);

    int result = system(command);
    
    return result;
} 

void* server_thread(void* args)
{
	int listenfd;
	struct sockaddr_in servaddr;

	listenfd = socket(PF_INET, SOCK_STREAM, 0);

	bzero(&servaddr, sizeof(servaddr));
	servaddr.sin_family = AF_INET;
	servaddr.sin_addr.s_addr = inet_addr(TUN_ADDR);
	servaddr.sin_port = htons(TUN_PORT);

	bind(listenfd, (struct sockaddr *)&servaddr, sizeof(servaddr));

	listen(listenfd, BACKLOG);
	while(1)
	{
		sleep(1);
	}

 	return NULL;
}

int server_setup()
{
	pthread_t thread;
	if (pthread_create(&thread, NULL, server_thread, NULL) != 0) {
		perror("pthread error");
		return -1;
	}	
}


void syn_send(int tun_fd, int i)
{
	char buffer[PCKT_LEN];
	struct iphdr *ip = (struct iphdr *) buffer;
	struct tcphdr *tcp = (struct tcphdr *)(buffer + sizeof(struct iphdr));
	uint16_t tot_len = sizeof(struct iphdr) + sizeof(struct tcphdr);
	uint16_t opt_len = 0;
	char* opt = (char*)(buffer + tot_len); // TCP option 
	memset(buffer, 0, PCKT_LEN);    

	if (spoof_mss[i] != 0)
	{
		struct mss_opt mss_opt;

		memset(&mss_opt, 1, sizeof(mss_opt));

		mss_opt.kind = 2;
		mss_opt.length = 4;
		mss_opt.mss = htons(spoof_mss[i]);

		memcpy(&opt[opt_len], &mss_opt, sizeof(mss_opt));  

		// if we have mss option
		tot_len += sizeof(mss_opt);
		opt_len += sizeof(mss_opt);
	}

	if (spoof_tsp[i] != 0)
	{
		struct tstamp_opt ts_opt;

		memset(&ts_opt, 1, sizeof(ts_opt));

		ts_opt.kind = 8;
		ts_opt.length = 10;
		ts_opt.tsval = htonl(spoof_tsp[i]);
		ts_opt.tsecr = 0;

		memcpy(&opt[opt_len], &ts_opt, sizeof(ts_opt));
		tot_len += sizeof(ts_opt);
		opt_len += sizeof(ts_opt);
	}

	if (spoof_wscale[i] != 0)
	{
		struct wscale_opt wscale_opt;

		memset(&wscale_opt, 1, sizeof(wscale_opt));

		wscale_opt.kind = 3;
		wscale_opt.length = 3;
		wscale_opt.scale = spoof_wscale[i];
		
		memcpy(&opt[opt_len], &wscale_opt, sizeof(wscale_opt));
		tot_len += sizeof(wscale_opt);
		opt_len += sizeof(wscale_opt);
	}

	ip->ihl = 5;
	ip->version = 4;
	ip->tos = 16;
	ip->tot_len = htons(tot_len);
	ip->id = htons(60000 + i);
	ip->frag_off = 0;
	ip->ttl = 64;
	ip->protocol = 6; // TCP 
	ip->saddr = inet_addr(spoof_ip_list[i]);
	ip->daddr = inet_addr(TUN_ADDR);
	ip->check = calc_cksm((unsigned short *)buffer,sizeof(struct iphdr));

	tcp->th_sport = htons(60000 + i);
	tcp->th_dport = htons(TUN_PORT);
	tcp->th_seq = htonl(1);
	tcp->th_ack = 0;
	tcp->th_off = (sizeof(struct tcphdr) + opt_len + sizeof(uint32_t) - 1) / sizeof(uint32_t);
	tcp->th_flags = TH_SYN;
	tcp->th_win = htons(4096);
	tcp->th_urp = 0;
	tcp->th_sum = 0;
	tcp->th_sum = tcp_checksum(ip, tcp, opt, opt_len);  

	if(write(tun_fd, buffer, tot_len) < 0)
	{
		perror("write() error");
		exit(-1);
	}
	else
	{
		printf("send packet %d\n", i);
	}

	return;
}

int main(int argc, char *argv[])
{
    int tun_fd, err;

    
    struct sockaddr_in sin, din;
    int one = 1;
    const int *val = &one;

    
    tun_fd = tun_create(IFF_TUN | IFF_NO_PI);
    if (tun_fd < 0)
    {
        perror("tun_create");
        return 0;
    }
    
    if (tun_setup("tun0") < 0)
    {
        perror("tun_setup");
        return 0;
    } 
    
    if (tun_setup("tun0") < 0)
    {
        perror("tun_setup");
        return 0;
    }

    if (tun_set_address("tun0", TUN_ADDR, 24) < 0)
    {
        perror("set address");
        return 0;
    }

    if (tun_set_route() < 0)
    {
        perror("set address");
        return 0;
    }

 	server_setup();

    sleep(5);

	for(int i = 0; i < COUNT; i++)
    {
        syn_send(tun_fd, i);
		
        usleep(10000);
    }

 	sleep(5);
	
    close(tun_fd);

    return 0; 
}

```
