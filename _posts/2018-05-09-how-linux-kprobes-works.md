---
layout    : post
title     : "[译] Linux 中的 Kprobe 是如何工作的?"
date      : 2018-05-09
lastupdate: 2018-05-09
categories: Translate
---

> 原文链接：https://vjordan.info/log/fpga/how-linux-kprobes-works.html
>

本文所有内容均来自内核 [kprobe文档](https://www.kernel.org/doc/Documentation/kprobes.txt). 

### 概要

Linux 中有 kprobe 和 jprobe. jprobe 是 kprobe 的特殊形式, 特别之处在于它可以获得函数参数。通过这个函数参数，我们可以过滤函数运行的轨迹。kprobe 还有一种优化版本--- optimized kprobe , 它可以避免 CPU 自陷入异常处理(trap)，这让它比普通 kprobe 运行速度快 10 倍。

### 正常的 kprobe

> Kprobe 是如何工作的 ？

当用户在一个**指令** (instruction) 上注册一个 kprobe 探针时，kprobe 会拷贝这指令，然后用一个**断点指令** (break instruction) 替换原来的指令 。在 i386 或者 x86_64 架构下，这个指令是**int3**.

当 CPU 运行过程中执行到这个断点时，会产生一次自陷 (trap), CPU 将此时寄存器的值保存起来，然后将控制权通过通知链 (notifier_call_chain) 机制交给 kprobe . kprobe 执行预先注册的 "pre_handler" 函数, 该函数的入参是 kprobe 控制块和保存的寄存器的值。

接下来，kprobe 会单步执行之前拷贝的原来那条指令的副本 (虽然在原处执行更轻松，但这需要 krpobe 临时移除设置的断点指令。这可能让其他 CPU 直接跳过探测点)

当原指令副本执行之后，kprobe 会执行预先注册的 "post_handler" 函数 (如果有的话)。

最后，探测点之后的原有指令被执行。
>

<p align="center"><img src="/assets/img/how-linux-kprobes-works/kprobe_diag1.png"></p>

注意: 图中的 trap 是指**中断3**(也是就是 int3). 详细信息请看该[文档](http://www.intel.com/Assets/en_US/PDF/manual/253668.pdf)

### jprobe

> jprobe 和 probe 类似，但是它要求探测点是放在一个函数的入口 (kprobe 可以放在函数的内部某条指令), 这个特性让 jprobe 可以无缝地访问函数的入参。jprobe 的 handler 函数比较特殊，它必须与原函数有相同的签名(相同的参数列表和返回值类型)，并且必须以 jprobe_return() 作为结尾。

当 CPU 运行到探测点时，内核自陷 (trap)，kprobe 将当前寄存器的值和此时的一段栈复制一份。随后 kprobe 将保存的寄存器中的 PC 指针修改为 jprobe 的 handler 处理函数 (译者注：修改前的 PC 指针为正常执行过程中的下一条指令) ，然后再从 kprobe 引起的自陷中恢复。这样本该执行原来的下一条指令，现在却进入了 jprobe 的 handler 处理函数的控制，并且此时寄存器与栈都和放置探测点的函数相同(译者注：handler 运行的环境和原函数相同，而且它们的参数列表、返回值也相同)  

在 handler 函数执行最后，通过调用 jprobe_return(),此时会恢复原来的栈和原指令，就像什么也没发生过一样。

按照惯例，被调用的函数可能包含一些参数(入参或本地变量，它们存放在栈上)，而 gcc 编译生成的代码可能修改栈空间。因此，kprobe 需要提前对栈进行拷贝，等到 jprobe 的处理函数执行完毕再重新恢复。krpobe 最多拷贝 MAX_STACK_SIZE 字节长度的栈空间(在 i386 上这个值为 64)

还需要注意的是，被放置探针的函数的参数可能通过栈传递，也有可能通过寄存器传递。jprobe 可以在任意一种情况下工作，因为它的处理函数和原函数的有相同的原型。

<p align="center"><img src="/assets/img/how-linux-kprobes-works/kprobe_diag2.png"></p>
>


### Optimized kprobe

>1.4 How Does Jump Optimization Work?

If your kernel is built with CONFIG_OPTPROBES=y (currently this flag
is automatically set 'y' on x86/x86-64, non-preemptive kernel) and
the "debug.kprobes_optimization" kernel parameter is set to 1 (see
sysctl(8)), Kprobes tries to reduce probe-hit overhead by using a jump
instruction instead of a breakpoint instruction at each probepoint.

1.4.1 Init a Kprobe

[...]

1.4.2 Safety Check

[...]

1.4.3 Preparing Detour Buffer

Next, Kprobes prepares a "detour" buffer, which contains the following
instruction sequence:
- code to push the CPU's registers (emulating a breakpoint trap)
- a call to the trampoline code which calls user's probe handlers.
- code to restore registers
- the instructions from the optimized region
- a jump back to the original execution path.

1.4.4 Pre-optimization

After preparing the detour buffer, Kprobes verifies that none of the
following situations exist:
- The probe has either a break_handler (i.e., it's a jprobe) or a
post_handler.
- Other instructions in the optimized region are probed.
- The probe is disabled.
In any of the above cases, Kprobes won't start optimizing the probe.
Since these are temporary situations, Kprobes tries to start
optimizing it again if the situation is changed.

If the kprobe can be optimized, Kprobes enqueues the kprobe to an
optimizing list, and kicks the kprobe-optimizer workqueue to optimize
it.  If the to-be-optimized probepoint is hit before being optimized,
Kprobes returns control to the original instruction path by setting
the CPU's instruction pointer to the copied code in the detour buffer
-- thus at least avoiding the single-step.
>

<p align="center"><img src="/assets/img/how-linux-kprobes-works/kprobe_diag3.png"></p>