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

- Chaincode 链码
- Channel
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
- Membership Service Provider MSP
- Membership Services

- Ordering Service 排序服务或共识服务
  - 将交易排序放入block的节点的集合。ordering service独立于peer流程之外，并以先到先得的方式为网络上所有的channel作交易排序。ordering service支持可插拔实现，目前默认solo单节点共识、kafka分布式队列和SBFT简单拜占庭容错。ordering service是整个网络的公用binding，包含与每个Member相关的加密材料。
- Peer 节点
  - 一个网络实体，维护ledger并运行Chaincode容器来对ledger执行read-write操作。peer由Member拥有和维护。
- Policy 策略
- Proposal 提案
  - 一种针对 channel中某peer的背书请求。每个proposal要么是chaincode Instantiate 要么是chaincode invoke。
- Transaction 交易
  - chaincode Instantiate 或者chaincode invoke。Invoke是从ledger中请求read-write set；Instantiate是请求在peer上启动chaincode容器。

