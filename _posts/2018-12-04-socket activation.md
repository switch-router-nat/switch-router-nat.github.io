---
layout    : post
title     : "socket activation"
date      : 2018-12-04
lastupdate: 2018-12-04
categories: Network(others)
---
这几天在研究[runc][1]的实现过程, 注意到它使用了的[go-systemd][2]的`socket activation`package, 出于好奇心,于是一探究竟.

## 服务的并行启动

> systemd是一套用来管理主机上各个Daemon运行的工具, 开发目标是提供更优秀的框架以表示系统服务间的依赖关系，并依此实现系统初始化时服务的**并行**启动.

上面提到的**并行**很有意思, 不相干的Daemon并行启动自然没有什么问题,但倘若Daemon B依赖于Daemon A,那么它就必须等到Daemon A完成启动后才能启动,这就变成了**串行**.如果避免这种串行呢? 这需要了解两个Daemon的依赖性的本质.通常来说,如果Daemon B依赖Daemon A,那么就说明Daemon B(Clinet)启动时会向Daemon A(Server)发起连接.由于Daemon A和Daemon B通常在一台主机上, 因此它们之间的C-S连接通常使用Unix Socket完成.

而`socket activation`的思想就是: Daemon B启动时其实并不需要Daemon A真正运行起来,它只需要Daemon A建立的socket处于listen状态就OK了. 而这个socket不必由Daemon A建立, 而是由systemd在系统初始化时就建立. 当Daemon B发起启动时发起连接,systemd再将Daemon A启动,
当Daemon A启动后,然后将socket`归还给`Daemon A. 这个过程如下图所示.
![activation][3]

## 例子

### Server端

> vim server.c

```c
#include <stdio.h>
#include <stdlib.h>
#include <sys/types.h>          
#include <sys/socket.h>
#include <unistd.h>
#include <sys/un.h>
#include <systemd/sd-daemon.h>

int main()
{
    int listenfd, connfd;
    char recvbuf[1024];
    ssize_t n;

    union {
       struct sockaddr sa;
       struct sockaddr_un un;
    }sa;

    if (sd_listen_fds(0)!=1) {
        fprintf(stderr,"No or too many fd received\n");
    }

    listenfd = SD_LISTEN_FDS_START + 0;

    if ((connfd = accept(listenfd, (struct sockaddr*)NULL, NULL))<0) {
        fprintf(stderr ,"accept(): %m\n");
    }     

    for (;;) {
        if ((n = read(connfd, recvbuf, sizeof(recvbuf))) < 0) {
            fprintf(stderr, "read()\n");
        }
       
        recvbuf[n] = '\0';
        write(connfd, recvbuf, n);
    }
    close(connfd);
    close(listenfd);
    return 0;
} 

```
其中`sd_listen_fds`将从systemd拿到处于listen状态的socket.

编译生成可执行文件, 并将其移动到/usr/bin/目录


> gcc server.c -lsystemd -o foobard
> mv foobard /usr/bin/

如果编译提示找不到libsystemd或者<systemd/sd-daemon.h>,请参考文末附录安装libsystemd

> vim /lib/systemd/system/foobar.socket

指定systemd要创建的unix socket路径为/run/foobar.sk
```
[Socket]
ListenStream=/run/foobar.sk

[Install]
WantedBy=sockets.target 
```

> vim /lib/systemd/system/foobar.service

```
[Service]
ExecStart=/usr/bin/foobard 
```

### Clinet端

> vim clinet.c

```c
#include <stdio.h>
#include <stdlib.h>
#include <sys/types.h>         
#include <sys/socket.h>
#include <unistd.h>
#include <sys/un.h>

int main()
{
    int fd;
    char sendbuf[1024];
    ssize_t n;

    union {
       struct sockaddr sa;
       struct sockaddr_un un;
    }sa;


    fd = socket(AF_UNIX, SOCK_STREAM, 0);
    if (fd < 0) {
        fprintf(stderr,"socket():%m\n");
        exit(1);
    } 
    memset(&sa, 0, sizeof(sa));
    sa.un.sun_family = AF_UNIX;
    
    strncpy(sa.un.sun_path, "/run/foobar.sk", sizeof(sa.un.sun_path));
  
    if (connect(fd, (struct sockaddr*)&sa, sizeof(sa))<0){
        fprintf(stderr, "connect()%m\n");
        exit(1);
    }
    printf("connect success !\n");
    for (;;) {
        fgets(sendbuf, 1024, stdin);
        if ((n = write(fd, sendbuf, strlen(sendbuf)))<0){
            exit(1);
        }
       
        if ((n = read(fd, sendbuf, sizeof(sendbuf))) < 0){
            exit(1);
        }
        sendbuf[n] = '\0';
        printf("%s\n", sendbuf);
        memset(sendbuf, 0, sizeof(sendbuf));
    }

    return 0;
} 

```

### 运行
启动socket

```powershell
# systemctl enable foobar.socket  
# systemctl start foobar.socket 
# ls /run/foobar.sk  
/run/foobar.sk
```
启动client
```bash
# ./client
connect success !
abc
abc
```
使用`ps`命令可看到foobard已经启动
```
# ps -aux | grep foobar
root     19480  0.0  0.0  23148  1360 ?        Ss   09:36   0:00 /usr/bin/foobard
```

## 附录 systemd 安装
在[官网][4]上下载安装包，这里以最新的[systemd-221][5]为例，下载后解压缩
```bash
# xz -d systemd-221.tar.xz 
# tar -xvf systemd-221.tar
```
进入systemd-221目录
```
# cd systemd-221
systemd-221#./configure
```
如果提示 configure: error: Your intltool is too old.  You need intltool 0.40.0 or later.
更新intltool(我使用的是ubuntu 16.04,其他linux发行版可选择自己的安装方式或源码安装)
```
# apt-get install intltool
```

如果提示configure: error: *** gperf not found，则
```bash
# apt-get install gerf
```
如果提示configure: error: *** libmount support required but libraries not found，则
```bash
# apt-get install libmount-dev
```

当通过configure检查之后，编译安装systemd
```bash
# make
# make install
```


## 参考资料

[systemd for developer][6] Pid Eins:systemd的使用方法
[sd_listen_fds][7] 


  [1]: https://github.com/opencontainers/runc
  [2]: https://github.com/coreos/go-systemd
  [3]: https://image-static.segmentfault.com/231/576/2315767774-5bf9ffa31385d_articlex
  [4]: https://www.freedesktop.org/software/systemd/
  [5]: https://www.freedesktop.org/software/systemd/systemd-221.tar.xz
  [6]: http://0pointer.de/blog/projects/socket-activation.html
  [7]: https://www.freedesktop.org/software/systemd/man/sd_listen_fds.html