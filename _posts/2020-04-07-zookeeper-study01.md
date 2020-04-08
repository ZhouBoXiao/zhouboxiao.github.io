---
layout:     post
title:      "Zookeeper相关知识01"
subtitle:   "待进一步完善"
date:       2020-04-07
author:     "ZBX"
header-img: "img/tag-bg.jpg"
tags:
    - Zookeeper
---



## ZAB协议

1. Leader election（选举阶段）：节点在一开始都处于选举阶段，只要有一个节点得到超过半数节点的票数，它就可以单选准leader。
2. Discovery（发现阶段）：在这个阶段，followers跟准Leader进行通信，同步followers最近接收的事务提议。
3. Synchronization（同步阶段）：同步阶段主要是利用leader前一阶段获得的最新提议历史，同步集群中所有的副本，同步完成之后准leader才会真正的leader。
4. Broadcast（广播阶段）到了这个阶段，Zookeeper集群才能正式对外提供事务服务，并且leader可以进行消息广播，同时如果有新的节点加入，还需要对新节点进行同步。