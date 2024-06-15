---
layout    : post
title     : "eventfd + SCM_RIGHTS 在进程间通信中的运用"
date      : 2020-03-28
lastupdate: 2020-03-28
categories: Kernel(others)
---

## eventfd 与进程间通信

Linux 环境下，不同进程间进行数据通信是一个十分常见的需求，通常我们可以使用 Unix Socket 很轻松的完成它。不过，如果传输的数据量比较大，那么使用共享内存或许是一个更好的解决方案。

一般地共享内存方案大概归结为以下几步：

1. 进程 A 将要共享的数据放到共享内存区域
2. 进程 A 通知进程 B 可以去共享内存区域读取数据
3. 进程 B 去共享内存区域读取数据

虽然我们还是绕不开进程间通信，但这么一分解，通知的数据量明显少了很多(只有第 2 步中的"踢一脚")，而 eventfd 恰好就足够了.

如果你翻看 eventfd 的[manpage](http://man7.org/linux/man-pages/man2/eventfd.2.html)的例子，你可能会产生一种误解：eventfd 只能用于父子进程(通过fork()产生)之间的通信，**这是不对的**。实际上 eventfd 可以用于任何两个进程的通信，造成误解的原因可能是觉得'文件描述符'的问题不好处理。

我们知道，文件描述符(file descripotr)是进程内的概念，不同进程的文件描述符是没有关系的。这一点在内核的角度也很容易理解，每个进程的 task struct 结构中都有一个 struct file\* 指针数组记录该进程打开了哪些'文件'，而文件描述符正是这个指针数组的下标，显然它是与进程相关的。

在 eventfd 的[manpage](http://man7.org/linux/man-pages/man2/eventfd.2.html) 中的例子中，由于 efd = eventfd(0,0) 是在 fork() 前调用的，所以父子进程对于 efd 的理解是一致的，因此可以分别读写该 eventfd 描述符。那么对于两个没有关系进程 A 和进程 B，它们如何完成有效的文件描述符传递呢？

## SCM_RIGHTS

**SCM_RIGHTS 可以解决这个问题**，看看 man 7 unix 的帮助信息：

```c 
Ancillary messages
       Ancillary  data  is  sent and received using sendmsg(2) and recvmsg(2).
       For historical reasons the ancillary message  types  listed  below  are
       specified with a SOL_SOCKET type even though they are AF_UNIX specific.
       To send them  set  the  cmsg_level  field  of  the  struct  cmsghdr  to
       SOL_SOCKET  and  the cmsg_type field to the type.  For more information
       see cmsg(3).

       SCM_RIGHTS
              Send or receive a set of  open  file  descriptors  from  another
              process.  The data portion contains an integer array of the file
              descriptors.  The passed file descriptors behave as though  they
              have been created with dup(2).
```

重点在于最后一句话，SCM_RIGHTS 传递的文件描述符表现的行为(behave)就像是接收进程使用 dup(2) 创建的一样。也就是说，SCM_RIGHTS 表面上传递的是文件描述符，但实际上并不是简单地传递描述符的数字，而是传递描述符背后的 file 文件。实际上，发送方和接收方的描述符数字是一样的可能性也是很小的......

在 cmsg 的[manpage](http://man7.org/linux/man-pages/man3/cmsg.3.html)中, 有使用 SCM_RIGHTS 的例子

```c 
struct msghdr msg = {0};
struct cmsghdr *cmsg;
int myfds[NUM_FD]; /* Contains the file descriptors to pass. */
char buf[CMSG_SPACE(sizeof myfds)];  /* ancillary data buffer */
int *fdptr;

msg.msg_control = buf;
msg.msg_controllen = sizeof buf;
cmsg = CMSG_FIRSTHDR(&msg);
cmsg->cmsg_level = SOL_SOCKET;
cmsg->cmsg_type = SCM_RIGHTS;
cmsg->cmsg_len = CMSG_LEN(sizeof(int) * NUM_FD);
/* Initialize the payload: */
fdptr = (int *) CMSG_DATA(cmsg);
memcpy(fdptr, myfds, NUM_FD * sizeof(int));
/* Sum of the length of all control messages in the buffer: */
msg.msg_controllen = cmsg->cmsg_len;
```

## REF 

[eventfd](http://man7.org/linux/man-pages/man2/eventfd.2.html)
[cmsg(3)](http://man7.org/linux/man-pages/man3/cmsg.3.html)
[know-your-scm_rights](https://blog.cloudflare.com/know-your-scm_rights/)
