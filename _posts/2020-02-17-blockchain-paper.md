---
layout:     post
title:      "BlockChain Security"
subtitle:   "区块链安全相关论文"
date:       2020-02-17
author:     "ZBX"
header-img: "img/tag-bg.jpg"
tags:
    - BlockChain
    - 翻译
---

## SmartPool

已经有超过区块链总计算能力40%的 mining pool 。这对分散化的本质构成了严重的威胁，使得区块链容易受到多种攻击。Loi等人提出了一种新的mining pool 系统SmartPool。SmartPool从Ethereum节点客户端(即，parity或geth)，其中包含mining任务信息。然后，miner 根据任务进行哈希计算，并将完成的shares返回给SmartPool客户机。当完成的shares数量达到一定数量时，它们将承诺使用部署在Ethereum中的SmartPool合约。SmartPool的合同将对shares进行核实，并向客户提供奖励。与传统P2P池相比，SmartPool系统具有以下优点：
1. 去中心化。SmartPool的核心以智能合约的形式实现，部署在区块链中。miner需要首先通过客户端连接到Ethereum和mine。mining pool 的运行可以依靠以太坊的共识机制。这样，它就确保了pool miners 的去中心化。mining pool 状态由Ethereum维护，不再需要pool operator
2. 效率。miners可以分批将完成的shares发送到smartpool合同。此外，miners只需要发送部分shares进行验证，而不是全部shares。因此，SmartPool比P2P池更有效。
3. 安全。SmartPool利用了一种新的数据结构，可以防止攻击者以不同的批次重新提交shares。此外，SmartPool的验证方法可以保证诚实的矿工即使在池中存在恶意的矿工，也能获得预期的奖励。

##  Quantitative Framework

区块链的性能和安全性之间存在权衡。Arthur等提出了一个定量框架，利用该框架分析基于PoW的区块链的执行性能和安全条款。如图11所示，该框架由两部分组成:区块链stimulator 和安全模型。该stimulator模拟区块链的执行，其输入为协议参数和网络参数。通过模拟器s分析，可以得到目标区块链的性能统计信息，包括块传播时间、块大小、网络延迟、过期块率、吞吐量等。陈旧块是指已挖掘但未写入公共链的块。吞吐量是区块链每秒可以处理的事务数。陈旧的块速率将作为参数传递给安全模型组件，该组件基于MDP(Markov决策过程)，用于击败双重开销和自私的挖掘攻击。该框架最终输出针对攻击的最优对抗策略，并为区块链构建安全条款提供便利。

## Oyente

Loi等人提出Oyente用于检测Ethereum智能合约中的bug。Oyente利用**符号执行**来分析智能合约的字节码，并遵循EVM的执行模型。由于Ethereum将智能合约的字节码存储在其区块链中，因此Oyente可以用于检测部署的合约中的bug。它以智能合约的字节码和Ethereum全局状态作为输入。首先，基于字节码，CFG BUILDER将静态构建智能合约的CFG(控制流图)。然后，根据Ethereum状态和CFG信息，EXPLORER利用静态符号执行对智能合约进行模拟执行。在此过程中，由于某些跳跃目标不是常数，CFG将得到进一步的丰富和完善;相反，应该在符号执行期间计算它们。核心分析模块使用相关的分析算法检测四个不同的漏洞(见3.2.2节)。验证器模块验证检测到的漏洞和脆弱路径。最终将确认的漏洞和CFG信息输出到VISUALIZER模块，用户可以使用该模块进行调试和程序分析。目前，Oyente是开放源码的公共使用[69]。

## Hawk
Ahmed等人提出了一种用于开发保护隐私的智能合同的新框架Hawk。利用Hawk，开发人员可以编写私有智能合约，并且不需要使用任何代码加密或混淆技术。此外，金融事务的信息不会显式地存储在区块链中。当程序员开发Hawk合约时，合约可以分为两部分:私有部分和公共部分。私有数据和财务功能相关代码可以写入私有部分，不涉及私有信息的代码可以写入公共部分。Hawk 允许程序员以直 观的方式编写具有隐私保护需求的智能合约,而不必考虑加密的实现,由 Hawk 编译器自动生成高效的基于==零知识证明==的加密协议来与区块链进行交互。Public 用来处理不涉及隐私数据的代码,Private用来处理隐私数据的代码,对Private 处理的数据进行加密保护。
 Hawk合同分为三部分。

1. 程序将在所有节点的虚拟机中执行，就像Ethereum中的智能合约一样。
2. 只由智能合同用户执行的程序。
3. 由manager执行的程序，是Hawk中特别值得信赖的一方。Hawk manager是在Intel SGX enclave 中执行的，它可以查看合同的隐私信息，但不会披露它。Hawk不仅可以保护隐私免受公众的侵害，还可以保护不同Hawk合同之间的隐私。如果管理器终止了Hawk协议，则会自动在财务上受到惩罚，用户将获得补偿。总的来说，Hawk可以在很大程度上保护用户在使用区块链时的隐私。


## Town Crier

智能合约通常需要与非链(即外部)数据源。Zhang等人提出了TC (Town Crier)，这是一种用于该数据交互过程的认证数据馈送系统。由于部署在区块链中的智能合约不能直接访问网络，所以它们不能通过HTTPS获取数据。TC正好充当支持http的数据源和智能合约之间的桥梁。TC的基本架构如图13所示。TC协议是TC系统的前端，充当用户协议与TC服务器之间的API。TC的核心程序在Intel SGX enclave中运行。TC服务器的主要功能是从用户的合约中获取数据请求，并从支持http的目标网站中获取数据。最后，TC服务器将以数字签名的区块链消息的形式向用户的合约返回一个数据报。TC可以在很大程度上保护数据请求过程的安全性。TC的核心模块分别运行在分散式Ethereum、支持sgx的enclave和支持http的网站上。此外，为了最大限度地提高安全性，enclave还禁用了网络连接功能。中继模块被设计成智能合同、SGX enclave环境和数据源网站的网络通信中心。实现了网络通信与TC核心程序执行的隔离。即使中继模块受到攻击，或者网络通信数据包被篡改，也不会改变TC的正常功能。TC系统为智能合同的离线数据交互提供了一个健壮的安全模型，并已作为公共服务在网上推出。



本文译自论文 A Survey on the Security of Blockchain Systems 
