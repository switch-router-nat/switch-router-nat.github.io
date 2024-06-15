---
layout    : post
title     : "以太坊源码分析—Ethash共识算法"
date      : 2018-04-26
lastupdate: 2018-04-26
categories: Blockchain
---

> Ethereum当前和Bitcoin一样，采用基于工作量证明(Proof of Work,PoW)的共识算法来产生新的区块。与Bitcoin不同的是，Ethereum采用的共识算法可以抵御ASIC矿机对挖矿工作的垄断地位，这个算法叫做`Ethash`。

#### 为什么要反ASIC
PoW的的核心是Hash运算，谁的Hash运算更快，谁就更有可能挖掘出新的区块，获得更多的经济利益。在Bitcoin的发展过程中，挖矿设备经历了(CPU=>GPU=>ASIC)的进化过程，其中的动机就是为了更快地进行Hash运算。随着矿机门槛地提高，参与者久越来越少，这与区块链的**去中心化**构想背道而驰。
因此，在共识算法设计时，为了减少ASIC矿机的优势(专用并行计算)，Ethereum增加了对于内存的要求，即在进行挖矿的过程中，需要占用消耗大量的内存空间，而这是ASIC矿机不具备的(配置符合运算那能力的内存太贵了，即使配置，这也就等同于大量CPU了)。即将挖矿算法从CPU密集型(CPU bound)转化为IO密集型(I/O bound)


#### Dagger-Hashimoto
`Ethash`是从`Dagger-Hashimoto`算法改动而来的，而`Dagger-Hashimoto`的原型是Thaddeus Dryja提出的[Hashimoto算法](http://diyhpl.us/~bryan/papers2/bitcoin/meh/hashimoto.pdf)，它在传统Bitcoin的工作量证明的基础上增加了消耗内存的步骤。

传统的PoW的本质是不断尝试不同的`nonce`，计算HASH
$$hash\_output=HASH(prev\_hash,merkle_root,nonce)$$
如果计算结果满足$hash\_output<target$，则说明计算的`nonce`是有效的

而对于Hashimoto，HASH运算仅仅是第一步，其算法如下:
```
nonce:　64-bits.正在尝试的nonce值
get_txid(T):历史区块上的交易T的hash
total_transactions:　历史上的所有交易的个数
```
```
hash_output_A = HASH(prev_hash,merkle_root,nonce)
for i = 0 to 63 do 
    shifted_A = hash_output_A >> i
    transaction = shifted_A mod total_transactions
    txid[i] = get_txit(transaction) << i
end of
txid_mix = txid[0]^txid[1]...txid[63]
final_output = txid_mix ^ （nonce<<192)
```
可以看出，在进行了HASH运算后，还需要进行64轮的混淆(mix)运算，而混淆的源数据是区块链上的历史交易，矿工节点在运行此算法时，需要访问内存中的历史交易信息(这是内存消耗的来源)，最终只有当　$final\_output < target$　时，才算是找到了有效的`nonce`

`Dagger-Hashimoto`相比于Hashimoto，不同点在于混淆运算的数据源不是区块链上的历史交易，而是以特定算法生成的约1GB大小的数据集合(`dataset`)，矿工节点在挖矿时，需要将这1GB数据全部载入内存。

#### Ethash算法概要

 - 矿工挖矿不再是仅仅将找到的`nonce`填入区块头，还需要填入一项`MixDigest`，这是在挖矿过程中计算出来的，它可以作为矿工的确在进行消耗内存挖矿工作量的证明。验证者在验证区块时也会用到这一项。
 - 先计算出约16MB大小的`cache`，约1GB的`dataset`由这约16MB的`cache`按特定算法生成，dataset中每一项数据都由`cache`中的256项数据参与生成，`cache`中的这256项数据可以看做是`dataset`中数据的`parent`。只所以是**约**，是因为其真正的大小是比16MB和1GB稍微小一点(为了好描述，以下将省略**约**)
 - `cache`和`dataset`的内容并非不变，它每隔一个`epoch`(30000个区块)就需要重新计算
 - `cache`和`dataset`的大小并非一成不变，16MB和1GB只是初始值，这个大小在每年会增大73%,这是为了抵消掉摩尔定律下硬件性能的提升，即使硬件性能提升了，那么最终计算所代表的工作量不会变化很多。结合上一条，那么其实每经过30000个区块，`cache`和`dataset`就会增大一点，并且重新计算
 - 全节点(比如矿工)会存储整个 `cache`和`dataset`，而轻客户端只需要存储 `cache`。挖矿(seal)时需要`dataset`在内存中便于随时存取，而验证(verify)时，只需要有cache就行，需要的`dataset`临时计算就行。

#### Ethash源码解析

##### dataset生成
`dataset`通过`generate()`方法生成，首先是生成cache，再从cache生成dataset

##### 挖矿(Seal)
在[挖矿与共识](https://segmentfault.com/a/1190000016994757)中提到了，共识算法通过实现`Engine.Seal`接口，来实现挖矿,Ethash算法也不例外。
其顶层流程如下:
![Seal](https://img-blog.csdn.net/20180715000627522?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2NoZW5tbzE4N0ozWDE=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

 - Seal调用中，启动一个go routine来调用`ethash.mine()`进行实际的挖矿，参数中的block是待挖掘的区块(已经打包好了交易)，而`nonce`是一个随机值，作为挖矿过程尝试`nonce`的初始值。
 - `mine()`调用首先计算后续挖矿需要的一些变量。**hash**为区块头中除了`nonce`和`mixdigest`的Hash值，**dataset**为挖掘这个区块时需要的混淆数据集合(占用1GB内存),**target**是本区块最终Hash需要达到的目标，它与区块难度成反比
 - 对本次尝试的`nonce`进行`hashmotoFull()`函数计算最终`result`(最终Hash值)和`digest`，如果满足target要求，则结束挖矿，否则增加`nonce`，再调用`hashmotoFull()`

```
func hashimotoFull(dataset []uint32, hash []byte, nonce uint64) ([]byte, []byte) {
	lookup := func(index uint32) []uint32 {
		offset := index * hashWords
		return dataset[offset : offset+hashWords]
	}
	return hashimoto(hash, nonce, uint64(len(dataset))*4, lookup)
}
```
`hashmotoFull()`是运算的核心，内部调用`hashmoto()`，第三个参数为`dataset`的大小（即1GB）,第四个参数是一个`lookup`函数，它接收**index**参数，返回`dataset`中64字节的数据。

```
func hashimoto(hash []byte, nonce uint64, size uint64, lookup func(index uint32) []uint32) ([]byte, []byte) {
	// 将dataset划分为2维矩阵，每行mixBytes=128字节，共1073739904/128=8388593行
	rows := uint32(size / mixBytes)
	
	//　将hash与待尝试的nonce组合成64字节的seed
	seed := make([]byte, 40)
	copy(seed, hash)
	binary.LittleEndian.PutUint64(seed[32:], nonce)

	seed = crypto.Keccak512(seed)
	seedHead := binary.LittleEndian.Uint32(seed)

	// 将64字节的seed转化为32个uint32的mix数组(前后16个uint32内容相同)
	mix := make([]uint32, mixBytes/4)
	for i := 0; i < len(mix); i++ {
		mix[i] = binary.LittleEndian.Uint32(seed[i%16*4:])
	}

	temp := make([]uint32, len(mix))

	//　进行总共loopAccesses=64轮的混淆计算，每次计算会去dataset里查询数据
	for i := 0; i < loopAccesses; i++ {
		parent := fnv(uint32(i)^seedHead, mix[i%len(mix)]) % rows
		for j := uint32(0); j < mixBytes/hashBytes; j++ {
			copy(temp[j*hashWords:], lookup(2*parent+j))
		}
		fnvHash(mix, temp)
	}
	// 压缩mix：将32个uint32的mix压缩成8个uint32
	for i := 0; i < len(mix); i += 4 {
		mix[i/4] = fnv(fnv(fnv(mix[i], mix[i+1]), mix[i+2]), mix[i+3])
	}
	mix = mix[:len(mix)/4]

	// 用8个uint32的mix填充32字节的digest
	digest := make([]byte, common.HashLength)
	for i, val := range mix {
		binary.LittleEndian.PutUint32(digest[i*4:], val)
	}
	//　对seed+digest计算hash，得到最终的hash值
	return digest, crypto.Keccak256(append(seed, digest...))
}
```
##### 验证(Verify)
验证时`VerifySeal()`调用`hashimotoLight()`，**Light**表明验证者不需要完整的dataset，它需要用到的dataset中的数据都是临时从cache中计算。
```
func hashimotoLight(size uint64, cache []uint32, hash []byte, nonce uint64) ([]byte, []byte) {
	keccak512 := makeHasher(sha3.NewKeccak512())

    //lookup函数和hashimotoFull中的不同，它调用generateDatasetItem从cache中临时计算
	lookup := func(index uint32) []uint32 {
		rawData := generateDatasetItem(cache, index, keccak512) //  return 64 byte

		data := make([]uint32, len(rawData)/4) //  16 个　uint32
		for i := 0; i < len(data); i++ {
			data[i] = binary.LittleEndian.Uint32(rawData[i*4:])
		}
		return data
	}

	return hashimoto(hash, nonce, size, lookup)
}
```
除了`lookup`函数不同，其余部分`hashimotoFull`完全一样


#### 总结
Ethash相比与Bitcoin的挖矿算法，增加了对内存使用的要求，要求矿工提供在挖矿过程中使用了大量内存的工作量证明,最终达到抵抗ASIC矿机的目的。

#### 参考资料
1 [Ethash-Design-Rationale](https://github.com/ethereum/wiki/wiki/Ethash-Design-Rationale)
2 [what-actually-is-a-dag](https://ethereum.stackexchange.com/questions/1993/what-actually-is-a-dag)
3 [why-dagger-hashimoto-for-ethereum](https://medium.com/verifyas/why-dagger-hashimoto-for-ethereum-773f0792a689)