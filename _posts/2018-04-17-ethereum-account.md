---
layout    : post
title     : "以太坊源码分析—账户的管理"
date      : 2018-04-17
lastupdate: 2018-04-17
categories: Blockchain
---

## 前言
以太坊是一个巨大的状态机，在网络中，每一个全节点都保存着以太坊状态机的全部历史，只要愿意，我们可以查询到任何时刻的`状态`(黄皮书中`World State`)，而账户状态便是其中的状态，这部分功能由主要由代码中的`state`包提供

## 基本概念

### 账户地址
![address.png](https://upload-images.jianshu.io/upload_images/12621947-5888105f58d3984c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

在以太坊中，无论是`外部账户`还是`合约账户`，都以一个`160bit`的数组表示地址，它是由特定椭圆曲线上的一个点表示的公钥经过`Keccak Hash`算法截取而来

关于椭圆曲线，请点击[椭圆曲线][1]
关于账户之间的区别，请点击[外部账户和合约账户的区别](https://ethfans.org/posts/479)

### 账户内容

以太坊中，一个账户用`Account`表示
```golang
type Account struct {
    Nonce      uint64
    Balance   *big.Int
    Root       common.Hash
    CodeHash   []byte
}
```
各个字段的意义如下：
* `Nonce`：账户发起交易的次数
* `Balance`：账户的余额
* `Root`<sup>[合约]</sup>：代表存储空间的一棵`MPT`树的根节点的`Hash`，可以简单地理解为一片存储空间，可以用它存储一些数据到区块链上，关于`MPT`，可以查看[这篇博文](https://blog.csdn.net/qq_33935254/article/details/55505472)。
* `CodeHash`<sup>[合约]</sup>：合约代码的Hash值
注：<sup>[合约]</sup>表示该项仅对合约账户有效

### 账户在区块链中的位置
![ether.png](https://upload-images.jianshu.io/upload_images/12621947-8648091b9ea81e98.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
所有账户以MPT树的形式组织起来，根节点的Hash值存储在区块`Header`的`stateRoot`字段

## 账户管理

### stateDB & stateObject
在以太坊账户管理中，`stateObject` 表示一个账户的`动态`变化，结构中的关键字段如下
```golang
type stateObject struct {
    address common.Address
    data   Account
    db     *StateDB
    trie   Trie
    code  Code
    ......
}
```
* `address` 为账户的160 bits 地址
* `data` 为账户的信息，即前面提到的**Account**结构
* `trie`   合约账户的存储空间的缓存，我们可以从由**data**的**Root**从底层数据库中读取这棵树，但鉴于我们会经常使用，所以把它缓存起来也不是一个坏主意
* `code`  合约代码的缓存，作用和trie类似       

`stateDB` 表示所有账户的`动态`变化，它管理`stateObject`，结构中的关键字段如下：
```golang
type stateDB struct {
    db    Database
    trie   Trie
    stateObjects  map[common.Address] * stateObject
    ......
}
```
* `db`   以太坊底层数据库接口，账户的信息都是从数据库中读取的
* `trie`  所有账户组织而成的的MPT树的实例，从它里面可以读取以太坊所有账户
* `stateObjects` 管理的所有**需要修改**的`stateObject`

## 账户操作
在执行区块中的交易时，我们可能需要修改某些账户的信息(比如增减余额，或者修改合约账户代码) ，这时我们按以下步骤进行操作
1. 从`stateDB`找到账户对应的`stateObject`，若不存在，则从`trie`树中，通过读取底层数据库构建新的`stateObject`，访问过的`stateObject`会缓存起来
2. 对`stateObject`账户进行操作，可能会涉及对余额的操作，如`AddBalance()`调用，也有可能对存储空间的操作，如`SetState()`，或者对合约代码的操作如`SetCode()`
3. 在区块构建完成时，计算每个账户新的`MPT`树的各个节点`Hash`，并存入数据库，完成修改。


  [1]: https://segmentfault.com/a/1190000017060251