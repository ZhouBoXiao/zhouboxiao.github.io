---
layout:     post   				    # 使用的布局（不需要改）
title:      Ethereum中的Bloom Filter			# 标题 
subtitle:   以太坊中的布隆过滤器 #副标题
date:       2020-02-24 				# 时间
author:     ZBX 						# 作者
header-img: img/post-bg-2015.jpg 	#这篇文章标题背景图片
catalog: true 						# 是否归档
tags:								#标签
    - Ethereum

---

## 概念

布隆过滤器（Bloom Filter）是1970年由布隆提出的。它实际上是一个很长的二进制向量和一系列随机映射函数。布隆过滤器可以用于检索一个元素是否在一个集合中。它的优点是空间效率和查询效率高，不会漏判，缺点是有一定的误判率。

## 日志中的Bloom Filter
以太坊提供Bloom Filter的功能是为了快速查找合约的交易或者发起交易的账户。某个账户根据合约地址查找相应的合约发生过的交易，就需要去每个区块头一个一个找到交易收据，Bloom Filter就可以快速的判断出来该块是否包含相关的记录。

下面是以太坊源码中区块头的结构的定义，可以看到中间有个Bloom的字段。
```go
type Header struct {
   ParentHash  common.Hash    `json:"parentHash"       gencodec:"required"`
   UncleHash   common.Hash    `json:"sha3Uncles"       gencodec:"required"`
   Coinbase    common.Address `json:"miner"            gencodec:"required"`
   Root        common.Hash    `json:"stateRoot"        gencodec:"required"`
   TxHash      common.Hash    `json:"transactionsRoot" gencodec:"required"`
   ReceiptHash common.Hash    `json:"receiptsRoot"     gencodec:"required"`
   Bloom       Bloom          `json:"logsBloom"        gencodec:"required"`
   Difficulty  *big.Int       `json:"difficulty"       gencodec:"required"`
   Number      *big.Int       `json:"number"           gencodec:"required"`
   GasLimit    uint64         `json:"gasLimit"         gencodec:"required"`
   GasUsed     uint64         `json:"gasUsed"          gencodec:"required"`
   Time        uint64         `json:"timestamp"        gencodec:"required"`
   Extra       []byte         `json:"extraData"        gencodec:"required"`
   MixDigest   common.Hash    `json:"mixHash"`
   Nonce       BlockNonce     `json:"nonce"`
}
```
Bloom的结构如下：
```go
const (
    // BloomByteLength represents the number of bytes used in a header log bloom.
    BloomByteLength = 256
    // BloomBitLength represents the number of bits used in a header log bloom.
    BloomBitLength = 8 * BloomByteLength
)
// Bloom represents a 2048 bit bloom filter.
type Bloom [BloomByteLength]byte

```
区块头中的Bloom字段是个2048的字节数组。

### 生成Bloom

下面是生成Bloom的CreateBloom方法的源码：
```go
func CreateBloom(receipts Receipts) Bloom {
    bin := new(big.Int)
    for _, receipt := range receipts {
        bin.Or(bin, LogsBloom(receipt.Logs))  //Logs中包含Address和Topics字段, Address是合约地址，Topics是合约的转账过程的账户地址
    }

    return BytesToBloom(bin.Bytes())
}
```
其中调用的LogsBloom方法如下：
```
func LogsBloom(logs []*Log) *big.Int {
    bin := new(big.Int)
    for _, log := range logs {
        bin.Or(bin, bloom9(log.Address.Bytes()))
        for _, b := range log.Topics {
            bin.Or(bin, bloom9(b[:]))
        }
    }

    return bin
}
```
该方法就是把Log中的Address和Topics放入bloom9中，bloom9方法如下：

```
func bloom9(b []byte) *big.Int {
    b = crypto.Keccak256(b)  // Keccak256是哈希算法，运算得160比特hash值
    r := new(big.Int)

    for i := 0; i < 6; i += 2 {
        t := big.NewInt(1)
        b := (uint(b[i+1]) + (uint(b[i]) << 8)) & 2047
        r.Or(r, t.Lsh(t, b))
    }

    return r
}
```
### BloomLookup查找
接下来就是检索元素是否在日志中
```
func bloomFilter(bloom types.Bloom, addresses []common.Address, topics [][]common.Hash) bool {
    if len(addresses) > 0 {
        var included bool
        for _, addr := range addresses {
            if types.BloomLookup(bloom, addr) { //首先查找Address
                included = true
                break
            }
        }
        if !included {
            return false
        }
    }

    for _, sub := range topics {      //如果包含相应的Address，则查找Topics
        included := len(sub) == 0 // empty rule set == wildcard
        for _, topic := range sub { 
            if types.BloomLookup(bloom, topic) {
                included = true
                break
            }
        }
        if !included {
            return false
        }
    }
    return true
}
```
其中BloomLookup方法如下：

```
func BloomLookup(bin Bloom, topic bytesBacked) bool {
    bloom := bin.Big()
    cmp := bloom9(topic.Bytes())

    return bloom.And(bloom, cmp).Cmp(cmp) == 0
}
```
