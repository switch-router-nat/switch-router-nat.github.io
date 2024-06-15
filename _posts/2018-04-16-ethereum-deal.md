---
layout    : post
title     : "以太坊源码分析—交易的执行"
date      : 2018-04-16
lastupdate: 2018-04-16
categories: Blockchain
---
## 前言
以太坊是一个运行智能合约的平台，被称作可编程的区块链，允许用户将编写的智能合约部署在区块链上运行。而运行合约的主体便是以太坊虚拟机(`EVM`)

### 区块 交易  合约
区块链由`区块`(`Block`)组成，而`区块`中打包一定数量的`交易`(`Transaction`)，`交易`可能是一个单纯的转账操作，也可能是调用一个智能合约，无论是哪一种，`EVM`在运行(`excute`)交易时都会创建`合约`(`Contract`)

### 外部账户 合约账户
以太坊中的账户有两类
* `外部账户`  由账户持有人的私钥控制的真实存在的账户
* `合约账户`  由合约代码控制，保存着合约代码
一笔`交易`总是有`发送方`(`sender`),`接收方`(`recipient`)和`数额`(`value`) 三要素。发送方将一定数额的`ETH`转移到接收方的账户，在单纯的转账交易中，接收方是外部账户。而在调用智能合约的交易时，接收方是合约账户。

### gas
如同现实中的税费一样，`交易`也需要将支付少量的费用，称为`gas`，费用支付给矿工，这可以激励矿工打包交易到区块，也使得区块链避免恶意运算攻击。`gas`由交易的发送者使用`ETH`购买，在执行交易的每一步都会消耗`gas`，如果`gas`用完了，交易状态会被回退，但消耗的`gas`不会返还。

## 交易执行
以太坊是一个基于交易的状态机，一笔交易可以使以太坊从一个状态(`state`)切换到另一个状态，即交易的执行伴随着状态的改变。
交易执行的入口在 `core/state_processor.go`的`Process()`方法，下面是该方法的轮廓 
```golang
func (p *StateProcessor) Process(block *types.Block, statedb *state.StateDB, cfg vm.Config) (types.Receipts，[]*types.Log，uint64，error) {
    ......
    var (
        usedGas = new(uint) 
        header = block.Header()
        gp = new(GasPool).AddGas(block.GasLimit())
    )
    for i, tx := range block.Transactions() {
        receipt, _, _ := ApplyTransaction(p.config, p.bc, nil, gp, statedb, header, tx, usedGas, cfg)
        receipts = append(receipts, receipt)
        allLogs = append(allLogs, receipt.Logs...)        
    }
    p.engine.Finalize(p.bc. header, statedb, block.Transactions(), block.Uncles(), receipts)
    ......
}
```
`Process()`方法对block中的每个交易`tx`调用`ApplyTransaction()`来执行交易，入参`state`存储了各个账户的信息，如账户余额、合约代码(仅对合约账户而言)，我们姑且将其理解为一个内存中的数据库。其中每个账户以`state object`表示

`ApplyTransaction()`方法完成以下功能
* 调用`AsMessage()`用`tx`为参数生成`core.Message`。也就是将`tx`中的一些字段存入`Message`，再从`tx`的数字签名中反解出`tx`的`sender`，重点关注其中的`data`字段：如果是普通的转账交易，该字段为空，如果是创建一个新的合约，该字段为新的合约的`代码`，如果是执行一个已经在区块链上存在的合约，该参数为合约代码的`输入参数`
* 调用`NewEVMContext()`创建一个`EVM`运行上下文`vm.Context`。注意其中的`Coinbase`字段需要填入的矿工的地址，`Transfer`是具体的转账方法，其实就是操作`sender`和`recipient`的账户余额
* 调用`NewEVM()`创建一个虚拟机运行环境`EVM`，它主要作用是汇集之前的信息以及创建一个代码解释器(`Interpreter`)，这个解释器之后会用来解释并执行合约代码
* 接下来就是调用`ApplyMessage()`将以上的信息**施加**在以太坊当前状态上，使得状态机发生状态变换

`ApplyMessage()`的顶层比较简单，它创建一个`StateTransition`结构并调用其`TransitionDb()`方法，`StateTransition`表示一次以太访的状态转移 其定义如下：
```golang
type StateTransition struct {
    gp  *GasPool
    msg Message
    gas  uint64
    gasPrice  *big,Int
    initialGas   uint64
    value   *big.Int
    data    []byte
    state   vm.StateDB
    evm    *vm.EVM
}
```
其中的字段都是之前`ApplyTransaction()`方法中创建的结构得到。一次状态转移包括以下流程
* `nonce`检查：交易的`nonce`值用于标识这是`sender`发起的交易的序号，该值总是等于上一笔交易的`nonce`值递增`1`，当我们检查发现当前Apply的这笔交易与该`sender`期待的`nonce`不一致时，就会拒绝此次状态转换
* `gas`预购：`sender`预购此次转换需要的`gas`，简单说来就是扣除`sender`账户的`ETH`(变化反映在`stateDB`)，扣除的数量却决于交易设定的`gasPrice`和`gasLimit`的乘积，单位是`gwei`。
*  合约账户创建： 如果交易的`recipient`为空的话，标识这笔交易需要创建一个合约，那么就创建一个合约账户(反映在`state object`)
* 价值转移：每笔交易都伴随着价值转移，即`ETH`从`sender`账户发送到`receipt`账户，如果创建了合约，还要执行合约代码

`TransitionDB()`完成这样的状态转换，其实现流程如下：
![TransitionDb.png](https://upload-images.jianshu.io/upload_images/12621947-896d4729d4d44cd9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

最终由交易的`receipt`是否为空决定是使用`evm.Create()`还是`evm.Call()`，无论是哪种，最终都是创建一个`Contract`结构，然后调用`run()`方法运行之。注意，即使是外部账户之间普通的转账也会调用`Call()`和`run()`，只是由于`receipt`上没有代码，运行会很快结束而已。`run()`最终调用`Interpreter`的`Run()`方法。

前面提到过，在调用`NewEVM()`时创建了一个解释器(`Interpreter`)
```
func NewInterpreter(evm *EVM，cfg Config)  *Interpreter {
     switch {
         case evm.ChainConfig().IsConstantinople(evm.BlockNumber):
             cfg.JumpTable = constantinopleInstructionSet
         case evm.ChainConfig().IsByzantium(evm.BlockNumber):
             cfg.JumpTable = byzantiumInstructionSet
         case evm.ChainConfig().IsHomestead(evm.BlockNumber):
             cfg.JumpTable = homesteadInstructionSet
         default:
             cfg.JumpTable = fromtierInstructionSet  
     }
     return &Interpreter{
         evm:      evm,
         cfg:      cfg,
         ......
     }
}
```
根据当前Block的高度，计算出它处于以太坊演进的阶段，得到该阶段支持的`指令集`(`InstructionSet`)，新的阶段在兼容老的阶段的所有指令前提下，再增加了独特的新指令。最终存储在`Interpreter`的`cfg`字段

**合约代码**本质上上是由`Solidity`语言编译后形成的`EVM字节码`,字节码中的操作也正是指令集中定义的指令

再回到`Run()`方法，其大概流程如下

![Run.png](https://upload-images.jianshu.io/upload_images/12621947-359d32a79feaffaa.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

`EVM`逐字节的解析合约代码并调用`excute()`方法运行，直到运行完成或者`gas`提前耗尽。

关于具体的`EVM`指令解释方式和虚拟机内部`栈`和`内存`等内部实现，参考[本系列文章](https://segmentfault.com/a/1190000017000539)
   
## 小结
1. 在以太坊中，交易的执行是由`EVM`完成的，网络中的所有全节点都会去执行每一笔交易(这样所有人的状态才可以保持一致)
2. 交易分为普通转账和执行(创建)智能合约，两者都由`sender`付费，后者相比前者，`EVM`要额外执行合约的字节码

