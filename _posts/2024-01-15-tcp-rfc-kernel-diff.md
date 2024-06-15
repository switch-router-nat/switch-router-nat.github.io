---
layout    : post
title     : "linux 内核实现与 rfc 不一致的地方"
date      : 2024-01-15
lastupdate: 2024-01-15
categories: Network(Kernel)
---

<mark>本文后续不定时更新</mark>

#### SYN 报文是否能携带 payload (不考虑 TCP FastOpen)

[TCP Fast Open(TFO)](https://switch-router.gitee.io/blog/tcp-fastopen/)

#### TCP ISN 递增速度

[理解 TCP 初始序号选择(ISN Selection)](https://switch-router.gitee.io/blog/tcp-isn/)

#### TCP TIME-WAIT 时间

[tcp_tw_reuse 的原理和实现](https://switch-router.gitee.io/blog/tcp-tw-reuse/)

#### TCP MSS 协商

[Linux内核协议栈中一些关于 TCP MSS 的细节](https://switch-router.gitee.io/blog/tcp-mss/)

#### rfc 接收端糊涂窗口综合征 vs 内核首部预测

[一个 TCP 接收缓冲区问题的解析](https://switch-router.gitee.io/blog/sk-rcvbuf/)


<mark>本文不定时更新</mark>