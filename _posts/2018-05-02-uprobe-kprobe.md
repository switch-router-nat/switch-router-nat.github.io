---
layout    : post
title     : "[译]动态跟踪Linux用户空间与内核空间中程序的运行过程"
date      : 2018-05-02
lastupdate: 2018-05-02
categories: Translate
---

> 原文链接：https://opensource.com/article/17/7/dynamic-tracing-linux-user-and-kernel-space
>

<p align="center"><img src="/assets/img/uprobe-kprobe/colors-colorful-box-rectangle.png"></p>
<p align="center">Image by : Internet Archive Book Images. Modified by Opensource.com. CC BY-SA 4.0</p>

你是否经历过这样一种情景：你的代码由于缺少print打印，这样你没办法判断出某一行代码是否被执行到，最终你只能加上打印信息后重新编译源文件? 现在有更轻松的解决方案了，那就是在你的代码(汇编指令)中添加动态探针(dynamic probe points)。

如果你是高级开发人员，那么就直接去看内核自带的[文档](https://www.kernel.org/doc/Documentation/trace/)或者[perf的帮助信息](http://man7.org/linux/man-pages/man1/perf.1.html)吧，它们介绍了多种不同的用户空间和内核空间跟踪机制。但如果你只是普通的使用者，只是想单纯地使用，那么本文一定会帮到你。

让我们先来熟悉一些概念吧~

#### 探测点(Probe point)

探测点指的是用于调试的语句，我们可以通过它窥测软件执行的行为(代码的执行流以及运行数据的值)。比如printk，它就是内核开发人员常常使用的办法。

#### 静态探测vs.动态探测

添加printk的方法属于静态探测，这是因为它的生效需要重新编译内核代码。除此之外，内核还内置了许多的跟踪点(tracepoint)，它们虽然可以被动态地使能或者去使能，但它们还是属于静态探测的范畴(因为它们的位置都是预设的，增删它们还是需要重新编译内核)。与之相反，内核还提供了一些框架，这些框架可以帮助开发者在不重新编译源码的情况下探测用户空间或者内核空间程序。[kprobe](https://www.kernel.org/doc/Documentation/trace/kprobetrace.txt)就是这样一种在内核代码中添加探测点的方法，而[uprobe](https://www.kernel.org/doc/Documentation/trace/uprobetracer.txt)则是在用户态。


### 使用uprobe跟踪用户空间程序

我们可以通过两种方法为用户空间程序添加uprobe探测点：其一是sysfs接口，其二是[perf tool](https://perf.wiki.kernel.org/index.php/Main_Page).

我将以下面这个简单的例子说明如何在特定的指令上添加探测点：

```c
/* test.c */
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

static int func_1_cnt;
static int func_2_cnt;

static void func_1(void)
{
        func_1_cnt++;
}

static void func_2(void)
{
        func_2_cnt++;
}

int main(int argc, void **argv)
{
	int number;

	while(1) {
		sleep(1);
		number = rand() % 10;
		if (number < 5)
			func_2();
		else
			func_1();
	}
}
```

编译代码，然后找到放置探测点的位置：
```
# gcc -o test test.c
# objdump -d test
```

假设现在我们得到了在ARM64平台上上面程序对应的指令:

```
0000000000400620 <func_1>:
  400620:       90000080        adrp    x0, 410000 <__FRAME_END__+0xf6f8>
  400624:       912d4000        add     x0, x0, #0xb50
  400628:       b9400000        ldr     w0, [x0]
  40062c:       11000401        add     w1, w0, #0x1
  400630:       90000080        adrp    x0, 410000 <__FRAME_END__+0xf6f8>
  400634:       912d4000        add     x0, x0, #0xb50
  400638:       b9000001        str     w1, [x0]
  40063c:       d65f03c0        ret

0000000000400640 <func_2>:
  400640:       90000080        adrp    x0, 410000 <__FRAME_END__+0xf6f8>
  400644:       912d5000        add     x0, x0, #0xb54
  400648:       b9400000        ldr     w0, [x0]
  40064c:       11000401        add     w1, w0, #0x1
  400650:       90000080        adrp    x0, 410000 <__FRAME_END__+0xf6f8>
  400654:       912d5000        add     x0, x0, #0xb54
  400658:       b9000001        str     w1, [x0]
  40065c:       d65f03c0        ret
```

现在我们希望在0x620和0x644的地方放置探测点，然后后台运行程序，于是执行下面的命令：

```
# echo 'p:func_2_entry test:0x620' > /sys/kernel/debug/tracing/uprobe_events
# echo 'p:func_1_entry test:0x644' >> /sys/kernel/debug/tracing/uprobe_events
# echo 1 > /sys/kernel/debug/tracing/events/uprobes/enable
# ./test &
```

前两条echo命令都是添加探测点，**p**表示它们是**简单探测点**(除此之外还有**返回探测点**)，func_n_entry 是之后trace输出时的名字，它是可选的，如果没有指定，则最后输出的名字会是p_test_0x644，其中test是二进制文件的名字。如果test不在当前目录，那么我们需要为其指定完整路径path_to_test/test。0x620和0x640是从程序起始处到对应指令的偏移量。请注意第二个echo后面跟的是>>，这代表一个追加添加操作。当我们最后使能uprobe时，以上两个探测点都会生效。当然我们也可以通过在events目录写特定event文件的方法来使能特定的event。一旦探测点被添加和使能，当相应指令执行，我们就能看到系统会生成对应的trace表项。

```
# cat /sys/kernel/debug/tracing/trace
# tracer: nop
#
# entries-in-buffer/entries-written: 8/8   #P:8
#
#                              _-----=> irqs-off
#                             / _----=> need-resched
#                            | / _---=> hardirq/softirq
#                            || / _--=> preempt-depth
#                            ||| /     delay
#           TASK-PID   CPU#  ||||    TIMESTAMP  FUNCTION
#              | |       |   ||||       |         |
            test-2788  [003] ....  1740.674740: func_1_entry: (0x400644)
            test-2788  [003] ....  1741.674854: func_1_entry: (0x400644)
            test-2788  [003] ....  1742.674949: func_2_entry: (0x400620)
            test-2788  [003] ....  1743.675065: func_2_entry: (0x400620)
            test-2788  [003] ....  1744.675158: func_1_entry: (0x400644)
            test-2788  [003] ....  1745.675273: func_1_entry: (0x400644)
            test-2788  [003] ....  1746.675390: func_2_entry: (0x400620)
            test-2788  [003] ....  1747.675503: func_2_entry: (0x400620)
```

我们可以通过上面的信息获取是执行到探测点的CPU以及对应的时刻。


**返回探测点**(return probe)可以被放置在任意一条指令上，当包含它的函数返回时，它会生成一条表项。

```
# echo 0 > /sys/kernel/debug/tracing/events/uprobes/enable
# echo 'r:func_2_exit test:0x620' >> /sys/kernel/debug/tracing/uprobe_events
# echo 'r:func_1_exit test:0x644' >> /sys/kernel/debug/tracing/uprobe_events
# echo 1 > /sys/kernel/debug/tracing/events/uprobes/enable
```

添加返回探测点时，我们使用**r**,而不是**p**，除此之外，其他参数与简单探测点是一样的。还有一点需要注意的是，如果我们希望添加新的探测点，我们必须先去使能当前的探测功能。

```
 test-3009  [002] ....  4813.852674: func_1_entry: (0x400644)
            test-3009  [002] ....  4813.852691: func_1_exit: (0x4006b0 <- 0x400644)
            test-3009  [002] ....  4814.852805: func_2_entry: (0x400620)
            test-3009  [002] ....  4814.852807: func_2_exit: (0x4006b8 <- 0x400620)
            test-3009  [002] ....  4815.852920: func_2_entry: (0x400620)
            test-3009  [002] ....  4815.852921: func_2_exit: (0x4006b8 <- 0x400620)
```

上面的输出日志告诉我们，func_1在时间戳4813.852691时返回到地址0x4006b0

```
# echo 0 > /sys/kernel/debug/tracing/events/uprobes/enable
# echo 'p:func_2_entry test:0x630' > /sys/kernel/debug/tracing/uprobe_events count=%x1
# echo 1 > /sys/kernel/debug/tracing/events/uprobes/enable
# echo > /sys/kernel/debug/tracing/trace
# ./test&
```

如上所示，我们还可以在程序执行到探测点时打印出ARM64 x1寄存器的值

```
   test-3095  [003] ....  7918.629728: func_2_entry: (0x400630) count=0x1
            test-3095  [003] ....  7919.629884: func_2_entry: (0x400630) count=0x2
            test-3095  [003] ....  7920.629988: func_2_entry: (0x400630) count=0x3
            test-3095  [003] ....  7922.630272: func_2_entry: (0x400630) count=0x4
            test-3095  [003] ....  7923.630429: func_2_entry: (0x400630) count=0x5
```

### 使用kprobe跟踪内核空间程序

kprobe和uprobe类似，用户可以通过sysfs接口在内核代码插入探测点。

我们可以将kprobe探测点放置在绝大部分内核符号上(/proc/kallsyms),除了处于黑名单中的符号(指添加了NOKPROBE_SYMBOL属性的符号)。和uprobe类似，探测点也可以放置在一个符号加上指定偏移的地方，或者是一个函数的返回点。并且，我们还可以将函数局部变量的值打印在输出中

下面即是例子：

```
; disable all events, just to insure that we see only kprobe output in trace.
# echo 0 > /sys/kernel/debug/tracing/events/enable
; disable kprobe events until probe points are inseted.
# echo 0 > /sys/kernel/debug/tracing/events/kprobes/enable
; clear out all the events from kprobe_events, to insure that we see output for
; only those for which we have enabled 
# echo > /sys/kernel/debug/tracing/kprobe_events
; insert probe point at kfree
# echo "p kfree" >> /sys/kernel/debug/tracing/kprobe_events
; insert probe point at kfree+0x10 with name kfree_probe_10
# echo "p:kree_probe_10 kfree+0x10" >> /sys/kernel/debug/tracing/kprobe_events
; insert probe point at kfree return
# echo "r:kfree_probe kfree" >> /sys/kernel/debug/tracing/kprobe_events
; enable kprobe events until probe points are inseted.
# echo 1 > /sys/kernel/debug/tracing/events/kprobes/enable
```


```
[root@pratyush ~]# more /sys/kernel/debug/tracing/trace
# tracer: nop
#
# entries-in-buffer/entries-written: 9037/9037   #P:8
#
#                              _-----=> irqs-off
#                             / _----=> need-resched
#                            | / _---=> hardirq/softirq
#                            || / _--=> preempt-depth
#                            ||| /     delay
#           TASK-PID   CPU#  ||||    TIMESTAMP  FUNCTION
#              | |       |   ||||       |         |
            sshd-2189  [002] dn..  1908.930731: kree_probe: (__audit_syscall_exit+0x194/0x218 <- kfree)
            sshd-2189  [002] d...  1908.930744: p_kfree_0: (kfree+0x0/0x214)
            sshd-2189  [002] d...  1908.930746: kree_probe_10: (kfree+0x10/0x214)
```
