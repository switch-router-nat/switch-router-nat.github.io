---
layout    : post
title     : "深入理解以太坊虚拟机 (一) [翻译]"
date      : 2018-04-29
lastupdate: 2018-04-29
categories: Blockchain
---

原文:  [Diving Into The Ethereum VM](https://blog.qtum.org/diving-into-the-ethereum-vm-6e8d5d2f3c30)
作者:  Howard
译者: 187J3X1　

Solidity offers many high-level language abstractions, but these features make it hard to understand what’s really going on when my program is running. Reading the Solidity documentation still left me confused over very basic things.

**Solidity提供了许多高级语言的特性，但这些高级特性使得要想去理解底层程序是如何运行过程变得困难。即使读了[Solidity的官方文档](https://solidity.readthedocs.io/en/develop/solidity-in-depth.html)，我依然对一些基础的内容感到困惑。**

What are the differences between *string*, *bytes32*, *byte[]*, *bytes*?
* Which one do I use, when?
* What’s happening when I cast a *string* to *bytes*? Can I cast to *byte[]*?
* How much do they cost?

***string*，*byte32*，*byte[]*，bytes这些类型到底有什么区别?**

 - **什么情景该选择使用哪一个?**
 - **将*string*类型转换为*bytes*类型时会发生什么? 能转换成 *byte[]*类型吗?**
 - **使用它们各有多大的代价?**
 
How are *mappings* stored by the EVM?
* Why can’t I delete a mapping?
* Can I have mappings of mappings? (Yes, but how does that work?)
* Why is there storage mapping, but no memory mapping?

**映射类型(*mappings*)在EVM中时被如何存储的?**

 - **能删除一个映射?**
 - **能创建映射的映射吗?**
 - **为什么只有存储空间的映射，没有内存空间的映射？**

How does a compiled contract look to the EVM?
* How is a contract created?
* What is a constructor, really?
* What is the fallback function?

**EVM眼中经过编译的合约是什么样子?**

 - **合约如何创建?**
 - **构造函数是什么?**
 - **fallback函数是什么？**

I think it’s a good investment to learn how a high-level language like Solidity runs on the Ethereum VM (EVM). For couple of reasons.
    Solidity is not the last word. Better EVM languages are coming. (Pretty please?)
    The EVM is a database engine. To understand how smart contracts work in any EVM language, you have to understand how data is organized, stored, and manipulated.
    Know-how to be a contributor. The Ethereum toolchain is still very early. Knowing the EVM well would help you make awesome tools for yourself and others.
    Intellectual challenge. EVM gives you a good excuse to play at the intersection of cryptography, data structure, and programming language design.

**我认为去学习Solidity这样的高级语言在以太坊虚拟机(EVM)中的运行过程是项非常棒的投资。至少有以下这些好处**:

 - **Solidity并不是终点。更好的EVM语言已经在路上了。**
 - **EVM是一个数据库引擎，要想理解智能合约是如何工作的，首先要理解合约数据是如何组织、存储、操作的。**
 - **可以成为贡献者。以太坊的工具链还非常新，深入理解EVM可以帮助你和他人实现一些很棒的工具。**
 - **可以提升思维。EVM能让你深入研究密码学、数据结构、程序设计。**

**译注**：Solidity是高级语言，定制的编译器可以将这种高级语言转化为EVM能理解的一串二进制编码，所以只要能生成这种二进制码，并不一定限定在Solidity。有点类似与JAVA利用JVM实现跨平台。

In a series of articles, I’d like to deconstruct simple Solidity contracts in order to understand how it works as EVM bytecode.

An outline of what I hope to learn and write about:

*   The basics of EVM bytecode.
*   How different types (mappings, arrays) are represented.
*   What is going on when a new contract is created.
*   What is going on when a method is called.
*   How the ABI bridges different EVM languages.

My final goal is to be able to understand a compiled Solidity contract in its entirety. Let’s start by reading some basic EVM bytecode!

**在接下来的一些列文章中，我会以一些简单的Solidity编写的合约为例展示其在EVM中是如何工作的。**

**以下是我希望学到的知识大纲:**

 -   **EVM字节码基础.**
 -   **不同数据结构的组织方式，比如映射和数组(*mapping*,*arrays*)**
 -   **合约创建时发生了什么.**
 -   **方法被调用时发生了什么.**
 -   **ABI是如何桥接不同的EVM语言的.**
**我的最终目标是能够完全理解智能合约的工作原理，让我们从EVM字节码开始吧**

This table of [EVM Instruction Set](https://gist.github.com/hayeah/bd37a123c02fecffbe629bf98a8391df) would be a helpful reference.

**你可以随时查看**[EVM支持的指令集](https://gist.github.com/hayeah/bd37a123c02fecffbe629bf98a8391df)**以获得帮助。**

**译注**：指令集对应于源码 core/vm/opcodes.go

###A Simple Contract

Our first contract has a constructor and a state variable:
**第一个合约的例子包含一个构造函数和一个状态变量**

```solidity
// c1.sol
pragma solidity ^0.4.11;

contract C {
    uint256 a;

    function C() {
      a = 1;
    }
}
```
Compile this contract with solc:
**使用solc来编译这个合约:**

```sol
$ solc --bin --asm c1.sol

======= c1.sol:C =======
EVM assembly:
    /* "c1.sol":26:94  contract C {... */
  mstore(0x40, 0x60)
    /* "c1.sol":59:92  function C() {... */
  jumpi(tag_1, iszero(callvalue))
  0x0
  dup1
  revert
tag_1:
tag_2:
    /* "c1.sol":84:85  1 */
  0x1
    /* "c1.sol":80:81  a */
  0x0
    /* "c1.sol":80:85  a = 1 */
  dup2
  swap1
  sstore
  pop
    /* "c1.sol":59:92  function C() {... */
tag_3:
    /* "c1.sol":26:94  contract C {... */
tag_4:
  dataSize(sub_0)
  dup1
  dataOffset(sub_0)
  0x0
  codecopy
  0x0
  return
stop

sub_0: assembly {
        /* "c1.sol":26:94  contract C {... */
      mstore(0x40, 0x60)
    tag_1:
      0x0
      dup1
      revert

auxdata: 0xa165627a7a72305820af3193f6fd31031a0e0d2de1ad2c27352b1ce081b4f3c92b5650ca4dd542bb770029
}

Binary:
60606040523415600e57600080fd5b5b60016000819055505b5b60368060266000396000f30060606040525b600080fd00a165627a7a72305820af3193f6fd31031a0e0d2de1ad2c27352b1ce081b4f3c92b5650ca4dd542bb770029
```
The number 6060604052... is bytecode that the EVM actually runs.
**最后生成的6060604052...便是EVM实际运行的字节码**

###In Baby Steps

Half of the compiled assembly is boilerplate that’s similar across most Solidity programs. We’ll look at those later. For now, let’s examine the unique part of our contract, the humble storage variable assignment:

###一步一步分析
**上面编译生成的汇编代码有一半都是大部分Solidity程序固定的框架，所以我们只需要关注我们合约中独特的部分，即对存储变量赋值的那部分。**

```sol
a = 1
```

This assignment is represented by the bytecode 6001600081905550. Let’s break it up into one instruction per line:
**该赋值语句转化成字节码后是6001600081905550。将其按指令分行展示**
```sol
60 01
60 00
81
90
55
50
```
The EVM is basically a loop that execute each instruction from top to bottom. Let’s annotate the assembly code (indented under the label tag_2) with the corresponding bytecode to better see how they are associated:

**EVM从上倒下依次执行每条指令。让我们将tag2以下代码与其对应的助记符联系起来看：**

```sol
tag_2:
  // 60 01
  0x1
  // 60 00
  0x0
  // 81
  dup2
  // 90
  swap1
  // 55
  sstore
  // 50
  pop
```

Note that 0x1 in the assembly code is actually a shorthand for push(0x1). This instruction pushes the number 1 onto the stack.

It still hard to grok what’s going on just staring at it. Don’t worry though, it’s simple to simulate the EVM line by line.

**注意: 0x1是push(0x1)的简化形式，它将数字1压栈。**
**到目前为止依旧不是很清楚，别担心！走读EVM字节码并没有想象中的那么困难。**

###Simulating The EVM

The EVM is a stack machine. Instructions might use values on the stack as arguments, and push values onto the stack as results. Let’s consider the operation add.

**EVM是基于栈的机器，指令读取栈上元素的值作为输入，并将运算结果压栈。以`add`指令为例：**

Assume that there are two values on the stack:
**假设现在栈上已经有了两个元素如下：**

```
[1 2]
```

When the EVM sees add, it adds the top 2 items together, and pushes the answer back onto the stack, resulting in:

**当EVM运行到`add`指令时，它将栈顶两个元素弹出，将其相加后的记过压栈，运算之后的栈变成了：**
```
[3]
```

In what follows, we’ll notate the stack with []:
**下文都以**`[]`**表示EVM运行过程中栈的状态**

```
// The empty stack
// 空栈
stack: []
// Stack with three items. The top item is 3. The bottom item is 1.
//一个包含3个元素的栈，栈顶元素是3,栈底元素是1．
stack: [3 2 1]
```
And notate the contract storage with {}:
**另外，使用**`{}`**表示EVM运行时存储器的状态：**

```
// Nothing in storage.
// 空的存储器
store: {}
// The value 0x1 is stored at the position 0x0.
//在0x0位置包含一个值为0x1的元素
store: { 0x0 => 0x1 }
```

**译注**：在以太坊源码中，数据结构Stack表示栈，Memory表示存储器。

Let’s now look at some real bytecode. We’ll simulate the bytecode sequence 6001600081905550 as EVM would, and print out the machine state after each instruction:

**下面我们来看真实的字节码。我们会模拟EVM执行字节码 6001600081905550并标识出每一步后机器的状态**

```
// 60 01: pushes 1 onto stack
// 60 01: 将 1 压栈
0x1
  stack: [0x1]

// 60 00: pushes 0 onto stack
// 60 01: 将 ０ 压栈
0x0
  stack: [0x0 0x1]

// 81: duplicate the second item on the stack
// 81: 将栈顶往下第２个元素复制一次放到栈顶
dup2
  stack: [0x1 0x0 0x1]

// 90: swap the top two items
// 90: 交换栈顶两个元素
swap1
  stack: [0x0 0x1 0x1]

// 55: store the value 0x1 at position 0x0
// This instruction consumes the top 2 items
// 55:将 0x1 保存在 0x0
// 这条指令弹出栈顶前2个元素
sstore
  stack: [0x1]
  store: { 0x0 => 0x1 }

// 50: pop (throw away the top item)
// 50: 弹出栈顶元素
pop
  stack: []
  store: { 0x0 => 0x1 }
```
The end. The stack is empty, and there’s one item in storage.
**最终，栈空了。存储器中包含一个元素**

What’s worth noting is that Solidity had decided to store the state variable `uint256 a` at the position `0x0`. It's perfectly possible for other languages to choose to store the state variable elsewhere.

**值得注意的是Solidity已经会将**`uint256 a`**固定在**`0x0`**位置，在其他高级语言中，我们可以主动指定其存储位置。**

In pseudocode, what the EVM does for `6001600081905550` is essentially:
**伪代码表示**`6001600081905550`**就是**
```
// a = 1
sstore(0x0, 0x1)
```
Looking carefully, you’d see that the dup2, swap1, pop are superfluous. The assembly code could be simpler:
**仔细观察，你会发现诸如**`dup2`, `swap1`, `pop`**这些指令都是多余的。汇编代码像下面这样就足够了**：

```
0x1
0x0
sstore
```

You could try to simulate the above 3 instructions, and satisfy yourself that they indeed result in the same machine state:
**模拟执行以上三条指令，你会发现和之前的那种方式相比，最后的结果是一样的。**

```
stack: []
store: { 0x0 => 0x1 }
```

### Two Storage Variables
### 两个存储变量

Let’s add one extra storage variable of the same type:
**在之前的例子的基础上增加一个相同类型的变量**
```
// c2.sol
pragma solidity ^0.4.11;</pre>

contract C {
    uint256 a;
    uint256 b;

function C() {
      a = 1;
      b = 2;
    }
}
```

Compile, focusing on `tag_2`:
**编译后，仅关注**`tag_2`：
```
$ solc --bin --asm c2.sol

// ... more stuff omitted
tag_2:
    /* "c2.sol":99:100  1 */
  0x1
    /* "c2.sol":95:96  a */
  0x0
    /* "c2.sol":95:100  a = 1 */
  dup2
  swap1
  sstore
  pop
    /* "c2.sol":112:113  2 */
  0x2
    /* "c2.sol":108:109  b */
  0x1
    /* "c2.sol":108:113  b = 2 */
  dup2
  swap1
  sstore
  pop
```

The assembly in pseudocode:
**汇编的伪码为：**

```
// a = 1
sstore(0x0, 0x1)
// b = 2
sstore(0x1, 0x2)
```

What we learn here is that the two storage variables are positioned one after the other, with `a` in position `0x0` and `b` in position `0x1`.
**可以看到，两个变量以此存储在存储器中**，`a` **在**`0x0` **而** `b` **在** `0x1`.

### Storage Packing
### 存储空间压缩

Each slot storage can store 32 bytes. It’d be wasteful to use all 32 bytes if a variable only needs 16 bytes. Solidity optimizes for storage efficiency by packing two smaller data types into one storage slot if possible.

**(存储器由很多个存储槽组成，)每个存储槽可以存放32字节的数据。如果一个变量只需要16字节的存储空间，但却让它占用一个完整的32字节空间，显然很浪费的。因此Solidity编译器会近可能地将两个小的数据类型放到一个存储槽中。**

Let’s change `a` and `b` so they are only 16 bytes each:
**将上面例子中的**`a` **和** `b`**定义成16字节**

```
pragma solidity ^0.4.11;

contract C {
    uint128 a;
    uint128 b;

function C() {
      a = 1;
      b = 2;
    }
}
```
Compile the contract:
**编译合约**
```
$ solc --bin --asm c3.sol

The generated assembly is now more complex:

tag_2:
  // a = 1
 0x1
stack: [0x1]
  0x0
stack: [0x1,0x1]
  dup1
stack: [0x0,0x0,0x1]
  0x100
stack: [0x100,0x0,0x0,0x1]
  exp
stack: [0x1,0x0,0x1]
  dup2
stack: [0x0,0x1,0x0,0x1]
  sload
stack: [0x0,0x1,0x0,0x1]
  dup2
stack: [0x1,0x0,0x1,0x0,0x1]
  0xffffffffffffffffffffffffffffffff
stack: [0xff..ff,0x1,0x0,0x1,0x0,0x1]
  mul
stack: [0xff..ff,0x0,0x1,0x0,0x1]
  not
stack: [0x0,0x0,0x1,0x0,0x1]
  and
stack: [0x0,0x1,0x0,0x1]
  swap1
stack: [0x1,0x0,0x0,0x1]
  dup4
stack: [0x1,0x1,0x0,0x0,0x1]
  0xffffffffffffffffffffffffffffffff
stack: [0xff..ff，0x1,0x1,0x0,0x0,0x1]
  and
stack: [0x1,0x1,0x0,0x0,0x1]
  mul
stack: [0x1,0x0,0x0,0x1]
  or
stack: [0x1,0x0,0x1] 
 swap1
stack: [0x0,0x1,0x1]
  sstore
stack: [0x1]
storage:{0x0 => 0x1}
  pop
stack: [0x0]
------------------------------------------------------------------------
  0x2
stack: [0x2]
  0x0
stack: [0x0，0x2]
  0x10
stack: [0x10,0x0,0x2]
  0x100
stack: [0x100,0x10,0x0,0x2]
  exp
stack: [0x100..00,0x0,0x2]
  dup2
stack: [0x0,0x100..00,0x0,0x2]
  sload
stack: [0x1,0x100..00,0x0,0x2]
  dup2
stack: [0x100..00,0x1,0x100..00,0x0,0x2]
  0xffffffffffffffffffffffffffffffff
stack: [0xff..ff,0x100..00,0x1,0x100..00,0x0,0x2]
  mul
stack: [0xff..ff00..00,0x1,0x100..00,0x0,0x2]
  not
stack: [0x00..00ff..ff,0x1,0x100..00,0x0,0x2]
  and
stack: [0x1,0x100..00,0x0,0x2]
  swap1
stack: [0x100..00,0x1,0x0,0x2]
  dup4
stack: [0x2,0x100..00,0x1,0x0,0x2]
  0xffffffffffffffffffffffffffffffff
stack: [0xff..ff,2,0x100..00,1,0,2]
  and
stack: [0x2,0x100..00,0x1,0x0,0x2]
  mul
stack: [0x200..00,0x1,0x0,0x2]
  or
stack: [0x200..01,0x0,0x2]
  swap1
stack: [0x0,0x200..01,0x2]
  sstore
stack: [0x2]
{0x0 => 0x200..01}
  pop
stack: []
```
The above assembly code packs these two variables together in one storage position (`0x0`), like this:
**上面的汇编码最终让两个变量存储在相同的位置**(`0x0`)

```
[         b         ][         a         ]
[16 bytes / 128 bits][16 bytes / 128 bits]
```
**译注**：我将每一步执行之后的栈的状态也显示了出来，尽管结果符合预期，但我也不明白为什么感觉绕了很大一圈。

The reason to pack is because the most expensive operations by far are storage usage:
*   `sstore` costs 20000 gas for first write to a new position.
*   `sstore` costs 5000 gas for subsequent writes to an existing position.
*   `sload` costs 500 gas.
*   Most instructions costs 3~10 gases.

By using the same storage position, Solidity pays 5000 for the second store variable instead of 20000, saving us 15000 in gas.

**将变量压缩存储在一起的原因是在区块链中存储操作是到目前为止最昂贵的操作了：**

 -   `sstore` **在一个新的位置存储要花费20000 gas。**
 -   `sstore` **在一个旧的位置存储要花费5000 gas。**
 -   `sload` **从一个位置读取花费500 gas。**
 -   **大多数指令花费** 3~10 **gas**。

**译注**：存储很贵！存储很贵！存储很贵。每条指令的花费在 core\vm\jumptable.go中指令表的gasCost函数获取

### More Optimization
### 更多的优化

Instead of storing `a` and `b` with two separate `sstore` instructions, it should be possible to pack the two 128 bits numbers together in memory, then store them using just one `sstore`, saving an additional 5000 gas.

You can ask Solidity to make this optimization by turning on the `optimize` flag:
**上面的例子中，为了存储**`a`**和**`b`**两个变量，我们使用了两次** `sstore` **指令。其实完全可以先将这两个128比特变量在内存中就打包成一个变量，然后调用一次** `sstore` **指令，这样足足可以省下5000 gas。**

```
$ solc --bin --asm --optimize c3.sol
```
Which produces assembly code that uses just one sload and one sstore:
**(显式地使用--optimize选项)生成的汇编码只使用了一次**`sload` **和** `sstore`

```
tag_2:
    /* "c3.sol":95:96  a */
  0x0
    /* "c3.sol":95:100  a = 1 */
  dup1
  sload
    /* "c3.sol":108:113  b = 2 */
  0x200000000000000000000000000000000
  not(sub(exp(0x2, 0x80), 0x1))
    /* "c3.sol":95:100  a = 1 */
  swap1
  swap2
  and
    /* "c3.sol":99:100  1 */
  0x1
    /* "c3.sol":95:100  a = 1 */
  or
  sub(exp(0x2, 0x80), 0x1)
    /* "c3.sol":108:113  b = 2 */
  and
  or
  swap1
  sstore
```

The bytecode is:
**最终生成的字节码为**
```
600080547002000000000000000000000000000000006001608060020a03199091166001176001608060020a0316179055
```
And formatting the bytecode to one instruction per line:
**将字节码按指令逐条显示**

**译注**：同样，我将其每一步的栈的状态显示出来
```
// push 0x0
60 00
stack: [0x0]
// dup1
80
stack: [0x0,0x0]
// sload
54
stack: [0x0,0x0]
// push17 push the the next 17 bytes as a 32 bytes number
70 02 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
stack: [0x200..00,0x0,0x0]

/* not(sub(exp(0x2, 0x80), 0x1)) */
// push 0x1
60 01
stack: [0x1,0x200000000000000000000000000000000,0x0,0x0]
// push 0x80 (32)
60 80
stack: [0x80,0x1,0x200000000000000000000000000000000,0x0,0x0]
// push 0x02 (2)
60 02
stack: [0x02,0x80,0x1,0x200000000000000000000000000000000,0x0,0x0]
// exp
0a
stack: [0x100000000000000000000000000000000,0x1,0x200000000000000000000000000000000,0x0,0x0]
// sub
03
stack: [0xFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF,0x200000000000000000000000000000000,0x0,0x0]
// not
19
stack: [0xFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF000000000000000000000000000000000,0x200000000000000000000000000000000,0x0,0x0]
// swap1
90
stack: [0x200000000000000000000000000000000,0xFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF000000000000000000000000000000000,0x0,0x0]
// swap2
91
stack: [0x0,0xFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF000000000000000000000000000000000,0x200000000000000000000000000000000,0x0]
// and
16
stack: [0x0,0x200000000000000000000000000000000,0x0]
// push 0x1
60 01
stack: [0x1, 0x0,0x200000000000000000000000000000000,0x0]
// or
17
stack: [0x1,0x200000000000000000000000000000000,0x0]

/* sub(exp(0x2, 0x80), 0x1) */
// push 0x1
60 01
stack: [0x1,0x1,0x200000000000000000000000000000000,0x0]
// push 0x80
60 80
stack: [0x80,0x1,0x1,0x200000000000000000000000000000000,0x0]
// push 0x02
60 02
stack: [0x2,0x80,0x1,0x1,0x200000000000000000000000000000000,0x0]
// exp
0a
stack: [0x100000000000000000000000000000000,0x1,0x1,0x200000000000000000000000000000000,0x0]
// sub
03
stack: [0xFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF,0x1,0x200000000000000000000000000000000,0x0]
// and
16
stack: [0x1,0x200000000000000000000000000000000,0x0]
// or
17
stack: [0x200000000000000000000000000000001,0x0]
// swap1
90
stack: [0x0,0x200000000000000000000000000000001]
// sstore
55
stack: []
storeage:{0x0 => 0x200..01}
```
There are four magic values used in the assembly code:
**上面的汇编代码中出现了4个幻数(常数)**


 -   0x1 (16 bytes), using lower 16 bytes
 -   **0x1 (16 字节), 存放在低16字节**
```
// Represented as 0x01 in bytecode
16:32 0x00000000000000000000000000000000
00:16 0x00000000000000000000000000000001
```

 -   0x2 (16 bytes), using higher 16bytes
 -   **0x2 (16 字节), 存放在高16字节**
```
// Represented as 0x200000000000000000000000000000000 in bytecode
16:32 0x00000000000000000000000000000002
00:16 0x00000000000000000000000000000000
```

 -   not(sub(exp(0x2, 0x80), 0x1))
```
// Bitmask for the upper 16 bytes
16:32 0xFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF
00:16 0x00000000000000000000000000000000
```

 -   sub(exp(0x2, 0x80), 0x1)
```
// Bitmask for the lower 16 bytes
16:32 0x00000000000000000000000000000000 
00:16 0xFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF
```
The code does some bits-shuffling with these values to arrive at the desired result:
**最终通过位运算组合成最终的结果**
```
16:32 0x00000000000000000000000000000002 
00:16 0x00000000000000000000000000000001
```

Finally, this 32bytes value is stored at position `0x0`.
**最后再把32字节的结果存储在位置**`0x0`

#### Gas Usage
#### Gas 使用量

> 60008054700**200000000000000000000000000000000**6001608060020a03199091166001176001608060020a0316179055

Notice that `0x200000000000000000000000000000000` is embedded in the bytecode. But the compiler could’ve also chosen to calculate the value with the instructions `exp(0x2, 0x81)`, which results in shorter bytecode sequence.
**注意，在前面的例子中，我们将**`0x200000000000000000000000000000000` **直接嵌入在了最终的字节码中。编译器本可以通过**`exp(0x2, 0x81)`**得到相同的结果，显然后者的字节码要短一些**。

But it turns out that `0x200000000000000000000000000000000` is a cheaper than `exp(0x2, 0x81)`. Let's look at the gas fees involved:
 *   4 gas paid for every zero byte of data or code for a transaction.
 *   68 gas for every non-zero byte of data or code for a transaction.

**但实际上，用**`0x200000000000000000000000000000000` **的方式更节省gas，我们可以计算下**

 *   **字节码中的每个值为0的字节花费是 4 gas.**
 *   **字节码中的每个值为非0的字节花费是 68 gas.**

Let’s compare how much either representation costs in gas.

*   The bytecode `0x200000000000000000000000000000000`. It has many zeroes, which are cheap.
*   **于是，使用** `0x200000000000000000000000000000000`**的方式，得益于它包含大量的0，于是实际上它更便宜。**
(1 * 68) + (16 * 4) = 196.

*   The bytecode `608160020a`. Shorter, but no zeroes.
*   **相比之下，**`608160020a`**更短，但由于没有0，实际会消耗更多的gas**
5 * 68 = 340.

The longer sequence with more zeroes is actually cheaper!
**结论就是：拥有更多的0的长字节码序列更加便宜**

### Summary

An EVM compiler doesn’t exactly optimize for bytecode size or speed or memory efficiency. Instead, it optimizes for gas usage, which is an layer of indirection that incentivizes the sort of calculation that the Ethereum blockchain can do efficiently.

We’ve seen some quirky aspects of the EVM:

*   EVM is a 256bit machine. It is most natural to manipulate data in chunks of 32 bytes.
*   Persistent storage is quite expensive.
*   The Solidity compiler makes interesting choices in order to minimize gas usage.

Gas costs are set somewhat arbitrarily, and could well change in the future. As costs change, compilers would make different choices.

### 总结
**EVM编译器并不会为了执行速度和内存效率优化代码，取而代之的是，它会将代码优化地使用更少的gas。**

**我们从之前的例子可以看出EVM的一些特性**

*   **EVM 是256比特位宽的机器，可以天然处理32字节宽度的数据**
*   **存储是非常昂贵的操作**
*   **Solidity编译器尽可能地优化代码gas的使用量**

**Gas的计算方式看上去有些武断，也许在以后计算方式会改变。如果指令的花费价格发生变化，编译器也会相应改变生成的代码。**

* * *