---
layout    : post
title     : "win-minmax(窗口中的最值)算法"
date      : 2020-02-11
lastupdate: 2020-02-11
categories: Network(Kernel)
---

### 一个问题

如果让你设计这样一个信号峰值采集系统，你会如何设计呢？

- 系统每 1s 都可能接收到外界发送的一定强度的一个信号，且信号的强度在 [1,100] 范围内
- 你需要提供一个查询服务，向调用者返回在过去 30s 内信号强度的最大值

最朴素暴力的办法是使用一个能容量为 30 的环形数组，将每秒收到的信号的强度值记录在里面(如果没收到就视为 0)，当有人查询时，遍历整个数组，找到最大的值然后返回，如 Figure-1 所示.`WRITE`表示新收到的值写入的位置。

<p align="center"><img src="/assets/img/win-minmax/rb.PNG"></p>
<p align="center">Figure.1</p>

这样的系统能用肯定是能用的，但这样**每次查询最大值时都要遍历整个缓冲区，效率实在不高**。

那么很自然地，我们可以记录下当前窗口中的最大值(用`MAX`表示其在缓冲区的位置)。这样当需要查询最大值时，直接将`MAX`所在位置的值返回就行，如 Figure-2 所示

<p align="center"><img src="/assets/img/win-minmax/rb-max.PNG"></p>
<p align="center">Figure.2</p>


这样，当我们收到新的值时，只需要比较新的值与`MAX`位置的值的关系，再决定是否更新`MAX`

<p align="center"><img src="/assets/img/win-minmax/update-max.PNG"></p>
<p align="center">Figure.3</p>


但有点麻烦的是，如果收到的值一直小于`MAX`位置的值，`WRITE`就会追赶上`MAX`，这时，我们就需要将`MAX`移到下一个最大值的地方，这就还是需要遍历整个数组。

<p align="center"><img src="/assets/img/win-minmax/update-max2.PNG"></p>
<p align="center">Figure.4</p>


也许你会说，还可以将第二大数值的位置记录下来(记为`SEC MAX`),这样在发生`WRITE`就会追赶上`MAX`时，可以很快将`SEC MAX`变成新的`MAX`。但这样又会引入新的问题：`SEC MAX`同样需要遍历整个数组。所以，这不是解决之道。

### win-minmax

`win-minmax`是较新的 Linux 内核中用来记录窗口极值的算法实现。BBR 拥塞控制算法可以它来获得窗口时间内最小 RTT。

`win-minmax`的实现很小巧，代码见[windowed min/max tracker](https://gitlab.freedesktop.org/panfrost/linux/blob/6131837b1de66116459ef4413e26fdbc70d066dc/lib/win_minmax.c)

它用固定的内存记录一个时间窗口内最值(the best)、次最值(**2nd best**)和再次最值(**3rd best**)，以及对应的时间戳。有一点特别的是，同时**始终保持**这三组值的时间从早到晚的顺序为：最值、次最值、再次最值。

以最大值为例，三组值的直观关系是

<p align="center"><img src="/assets/img/win-minmax/minmax.PNG"></p>
<p align="center">Figure.5</p>


这样，这样当某个时刻收到新的值时，就根据值的大小更新这三组值。如果此时距收到最大值的时刻还不到一个窗口，就可根据新收到的值与已有的三个值的关系，分为以下四种情况：


<p align="center"><img src="/assets/img/win-minmax/4case.PNG"></p>
<p align="center">Figure.6</p>

总的思想就是，如果新值超过了已有的三个值的一项或者多项，就将超过的"合并"起来。比如，对于第一种情况，新值超过了三个值，那么此时就可以将这三组值归一。通过这种方式，`win-minmax`可以保持三组值的大小和时间关系。

需要注意的是，为了尽量让三组值尽量**widely separated in the time window**，算法限制除了重叠情况外,**2nd best** 的时间戳不会在 **1st best** 时间戳的 1/4 个窗口内，**3rd best** 的时间戳不会在 **1st best** 时间戳的 1/2 时间窗口内。 

`win-minmax`的优点显而易见：

- 使用与窗口大小无关的内存空间(无论窗口多大，都是三组值)
- 无需进行遍历数组操作，可以直接取得最值。

