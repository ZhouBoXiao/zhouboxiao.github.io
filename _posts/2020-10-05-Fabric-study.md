---
layout:     post
title:      "Fabric"
subtitle:   "Fabric相关知识"
date:       2020-10-05
author:     "ZBX"
header-img: "img/tag-bg.jpg"
tags:
    - BlockChain

---

## 词汇表

- **Chaincode** 链码，运行在节点内的程序，提供业务逻辑接口，对账本进行查询或更新
- Channel  通道，私有的子网络，通道中的节点共同维护账本，实现数据的隔离和保密。 每个channel对应一个账本，由加入该channel的peer维护，一个peer可以加入多个channel，维护多个账本。 通道，私有的子网络，通道中的节点共同维护账本，实现数据的隔离和保密。 每个channel对应一个账本，由加入该channel的peer维护，一个peer可以加入多个channel，维护多个账本。
- Commitment
- Current State
- Concurrency Control Version Check
- Anchor Peer 锚节点
- **Endorsement** 背书
  - chaincode具有相应的endorsement policies，其中指定了endorsing peer、
- **Edorsement policy**

- Fabric-ca 证书节点
- Genesis Block 创世区块
- **Gossip Protocol** 
  - 数据传输协议有三项功能：
    - 管理peer发现和channel成员

- **Leading Peer** 主导节点
  - 每一个Member在其订阅的channel上可以拥有多个peer，其中一个peer会作为channel的leading peer代表该Member与ordering service通信。ordering service将block传递给leading peer，该peer再将此block分发给同一member下的其他的peer。

- Ledger 账本
  - Ledger是个channel的chain和由channel中每个peer维护的数据库。
- Member 成员 
  - 拥有网络唯一根证书的合法独立实体。像peer节点和app client这样的网络组件会链接到一个Member。
- Membership Service Provider（MSP）成员管理服务，基于PKI实现，为网络成员生成证书，并管理身份
- Membership Services

- Ordering Service 排序服务或共识服务
  - 将交易排序放入block的节点的集合。ordering service独立于peer流程之外，并以先到先得的方式为网络上所有的channel作交易排序。ordering service支持可插拔实现，目前默认solo单节点共识、kafka分布式队列和SBFT简单拜占庭容错。ordering service是整个网络的公用binding，包含与每个Member相关的加密材料。
- Peer 节点
  - 一个网络实体，维护ledger并运行Chaincode容器来对ledger	执行read-write操作。peer由Member拥有和维护。
- Policy 策略
- Proposal 提案
  - 一种针对 channel中某peer的背书请求。每个proposal要么是chaincode Instantiate 要么是chaincode invoke。
- Public Key Infrastructure（PKI），一种遵循标准的利用公钥加密技术为电子商务的开展提供一套安全基础平台的技术和规范
- Transaction 交易
  - chaincode Instantiate 或者chaincode invoke。Invoke是从ledger中请求read-write set；Instantiate是请求在peer上启动chaincode容器。

## 交易流程

1. 发送交易提案
   客户端发送交易提案（Proposal）到背书节点，提案中包含交易所需参数。

2. 模拟执行交易提案
   背书节点会调用链码模拟执行交易提案(Proposal)，这些执行不会更新账本。每个执行都会产生对状态数据读出和写入的数据集合，叫做读写集（RWsets），读写集是交易中记录的主要内容。

3. 返回提案响应
   背书节点会对读写集进行背书(Endorse)签名，生成提案响应(Proposal response)并返回给应用程序。

4. 交易排序
   应用程序根据接收到的提案响应生成交易，并发送给排序服务节点。排序服务打包一组交易到一个区块后，分发给各记账节点。

5. 交易验证并提交
   每个节点会对区块中的所有交易进行验证，包括验证背书策略以及版本冲突验证（防止双花），验证不通过的交易会被标记会无效（Invalid）。账本更新：节点将读写集更新到状态数据库 ，将区块提交到区块链上。

6. 通知交易结果给客户端
   各记账节点通知应用程序交易的成功与否，交易完成。