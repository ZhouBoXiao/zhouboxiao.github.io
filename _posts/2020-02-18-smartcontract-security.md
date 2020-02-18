---
layout:     post   				    # 使用的布局（不需要改）
title:      Challenges for Combining Smart Contracts with Trusted Computing 				# 标题 
subtitle:   将智能合约与可信计算相结合的问题和挑战 #副标题
date:       2020-02-18 				# 时间
author:     ZBX 						# 作者
header-img: img/post-bg-2015.jpg 	#这篇文章标题背景图片
catalog: true 						# 是否归档
tags:								#标签
    - Smart Contract
    - SGX
    - 翻译
---



# Challenges for Combining Smart Contracts with Trusted Computing

**Author： Marcus Brandenburger、Christian Cachin 等**

## 1 INTRODUCTION

​		可信执行技术被设想在区块链和分布式账本上下文中发挥重要作用，特别是对于企业应用程序和联盟区块链。区块链上的智能合约不能保持secret，因为它的数据被复制到网络中的所有节点上。为了解决这个问题，有人建议[2,4,9]使用可信执行环境(TEE)，如Intel SGX[1,5,8]，来执行需要隐私的智能合约。因此，不受信任的区块链节点无法访问TEE中的数据和计算。密码协议(如安全多方计算或零知识证明)为区块链上的隐私提供了另一种解决方案，但是，它们还不够成熟，不能方便地运行通用计算。一个众所周知的例子是，TEE特别有用，它涉及通过区块链上的智能合约实现的投票。所有的个人投票都必须保密，只有投票结果才能公开。这可以通过对选票进行加密来实现，加密的方式是只有在TEE中运行的合约才能在计算票数时解密选票。投票应用程序是如何利用区块链的可验证性属性的一个主要示例，在该属性中，每个参与者都应该能够验证他们的投票在最终评估中得到了考虑。因此，智能合约可以对所有已统计的选票生成加密签名，从而验证投票过程。最近在TEE内执行智能合约的系统都有一个共同的目标，即保护合约的完整性和机密性。但是，它们在细节上有所不同，例如，它们支持不同的编程语言，包括c++、Java、EVM和Scheme。这些系统可以分为on-chain执行系统、Coco框架[9]和安全链代码执行系统(SCE-Fabric)[2]，以及执行契约off-chain执行系统，如超分类帐私有数据对象(PDO)项目[6]和Ekiden[3]。然而，在TEE中执行智能合约也会带来一些挑战。例如，由于TEEs通常是无状态的，因此很容易受到回滚攻击，这可能会导致丢失机密性，Brandenburger等人曾讨论过这一点。在本文的其余部分中，我们将讨论on-chain执行和off-chain执行的概念差异，以及智能合约执行与TEE结合所带来的技术挑战。

## 2 ON-CHAIN VS OFF-CHAIN EXECUTION

​		许多提议的系统使用off-chain execution[3,6]在TEE中运行智能合约。与on-chain执行不同，contract是由配备了TEE的特殊节点执行的，而不是由区块链节点执行的。这种方法的主要好处是它允许使用分类账抽象，因此可以支持不同的区块链系统。例如，Ekiden[3]演示了Ethereum和扩展Tendermint的区块链系统的实例化。然而，off-chain执行也将合约生命周期管理的责任委托给了基于TEE的智能合约系统。由于这个原因，PDO[6]维护一个合约注册表，而这个功能通常由区块链系统提供。

​		相反，在on-chain执行[2,9]将基于TEE的运行时转移到区块链节点。的确，这需要更多的集成工作，但正如我们现在所讨论的，这也带来了一些优势。智能合约实现有状态功能，该功能接收一些参数并相应地更新其状态。特别是，合约状态通常由多个数据项组成，这允许根据合约逻辑对部分状态进行细粒度访问。通过on-chain执行，智能合约可以直接访问本地分类账。Coco[9]在每个区块链节点上维护TEE中的本地分类账，而SCE-fabric[2]只在TEE中保存一些小型的完整性元数据，这些元数据用于在从本地分类账加载时验证状态。对于-off-chain执行，客户端必须在调用智能合约之前从分类帐中获取合约状态。这可能很困难，因为它需要知道执行需要哪些数据项。PDO[6]通过将合约状态封装在单个对象中简化了这一点。但是，对于较大的状态和并发执行，这种解决方案可能不能令人满意。回想一下投票示例，其中用户可以并发地投票。将单个状态对象用于整个投票应用程序将导致许多账本的读写冲突，因此客户必须重新提交他们的投票。

​		许多区块链还支持合约到合约的调用，允许一个智能合约调用另一个智能合约。因此，可以很容易地从许多合约组成复杂的应用程序。但是，如前所述，使用off-chain执行时，clients可能需要获取相应的状态，当涉及到contract-to-contract的执行时，这种状态可能更加复杂。在on-chain模型中，这不是问题。如上所述，选择on-chain vs off-chain可信执行有很多含义。根据用例、隐私需求和性能需求，哪个选项是首选的?

## 3 ATTESTATION AND KEY MANAGEMENT

​		认证允许客户验证正确的智能合约是在一个真正的TEE中加载和执行的。这对于在可能不受信任的节点上的TEE中执行的智能合约中建立信任至关重要。然而，认证是昂贵的，因为它通常需要一个交互式协议。例如，Intel SGX[1]的认证协议涉及一个可信的第三方(即，英特尔认证服务)。基于这个原因，所提议的系统[2,3,6,9]使用区块链为智能合同工构建认证基础设施。他们只在初始阶段进行认证，并将认证结果存储在分类账上。不幸的是，认证目前只被英特尔SGX等一小部分TEE所支持。为了防止TEE供应商锁定并提供互操作性，业界有必要采用一组常见的安全特性，包括认证，这也是Keystone-enclave项目[7]所建议的。一般情况下，TEE不保证可用性，即恶意节点可以随时终止智能合约TEE。为了克服这个问题，多个节点可以在每个节点上的一个TEE内执行相同的合约。然而，这需要密钥管理，这是另一个技术挑战。详细地说，合约状态仅在TEE中以明文形式存在，并在存储到分类帐时加密。不幸的是，由Intel SGX支持的数据密封不能直接用于跨多个TEE实例共享数据。一种常见的方法是建立用于加密的共享密钥。例如，Ekiden[3]使用一个特殊的密钥管理器TEE复制所有节点之间的每个智能契约的共享密钥，假设始终至少有一个节点可用。相反，PDO[6]使用基于阈值加密的分布式密钥供应服务。所提出的系统还没有显示出满意的解决方案的关键撤销，在关键妥协。具有多个TEE节点的智能合约执行的可用性和安全性之间的权衡是什么?

## 4 NON-DETERMINISTIC EXECUTION

​		一个普遍的观点是，通过在TEE中执行智能合约，区块链可以支持非确定性事务，正如Coco框架[9]所具有的特性。非确定性事务通常包含随机性或来自off-chain服务(oracle)的输入，例如TownCrier[10]。当事务仅由一个TEE执行，并且所有区块链节点都应用输出，因为执行是可信的时，这就可以工作了。然而，在实践中，**TEE可能会受到损害**，这将导致机密性的丧失，最显著的是，执行完整性不再得到保证。要解决这个问题，用于可验证执行或简单复制的密码协议可能有助于检测差异，并可能证明某个智能合约TEE的行为不正确。如何区分差异和非确定性执行?除了包含处理非确定性执行的方法的Hyperledger Fabric[2]之外，目前的系统没有解决这个问题。

### 参考文献

[1] Ittai Anati, Shay Gueron, Simon Johnson, and Vincent Scarlata. 2013. Innovative Technology for CPU Based Attestation and Sealing. In Int.Workshop on Hardware and Architectural Support for Security and Privacy (HASP). ACM.

[2] Marcus Brandenburger, Christian Cachin, Rüdiger Kapitza, and Alessandro Sorniotti. 2018. Blockchain and Trusted Computing: Problems, Pitfalls, and a Solution for Hyperledger Fabric. ArXiv e-prints (2018). https://arxiv.org/abs/1805.08541

[3] Raymond Cheng, Fan Zhang, Jernej Kos, Warren He, Nicholas Hynes, Noah M.Johnson, Ari Juels, Andrew Miller, and Dawn Song. 2018. Ekiden: A Platform for Confidentiality-Preserving, Trustworthy, and Performant Smart Contract Execution. ArXiv e-prints (2018). http://arxiv.org/abs/1804.05141

[4] Mike Hearn. 2016. Corda: A distributed ledger. (2016). Whitepaper, https://docs.corda.net/_static/corda-technical-whitepaper.pdf.

[5] Matthew Hoekstra, Reshma Lal, Pradeep Pappachan, Vinay Phegade, and Juan Del Cuvillo. 2013. Using Innovative Instructions to Create Trustworthy Software Solutions. In Int. Workshop on Hardware and Architectural Support for Security and Privacy (HASP). ACM.

[6] Intel. 2018. Private Data Objects (PDO). ArXiv e-prints (2018). https://arxiv.org/abs/1807.05686

[7] Keystone Project. 2018. https://keystone-enclave.org/.

[8] Frank McKeen, Ilya Alexandrovich, Alex Berenzon, Carlos V. Rozas, Hisham Shafi,Vedvyas Shanbhogue,  and Uday R. Savagaonkar. 2013. Innovative Instructions and Software Model for Isolated Execution. In Int. Workshop on Hardware and Architectural Support for Security and Privacy (HASP). ACM.

[9] Microsoft. 2017. The Coco Framework. (2017). Whitepaper, https://github.com/ Azure/coco-framework.

[10] Fan Zhang, Ethan Cecchetti, Kyle Croman, Ari Juels, and Elaine Shi. 2016. Town Crier: An Authenticated Data Feed for Smart Contracts. In Proc. ACM SIGSAC Conference on Computer and Communications Security (CCS).