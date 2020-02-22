---
layout:     post   				    # 使用的布局（不需要改）
title:      LLVM  Basic Block frequency			# 标题 
subtitle:   基本块频率 #副标题
date:       2020-02-22 				# 时间
author:     ZBX 						# 作者
header-img: img/post-bg-2015.jpg 	#这篇文章标题背景图片
catalog: true 						# 是否归档
tags:								#标签
    - LLVM
---

### **介绍**	

​	看了写论文中评价Fuzzer性能有一项关键性指标，即代码覆盖率（coverage），而Anti-fuzz的论文中则是提出了降低代码覆盖率的方案，而与覆盖率息息相关的是基本块频率，只有热的基本块的访问频率变低才能说明Anti-fuzz确实是有用的，于是查看了下LLVM中关于基本块频率的描述。

​		所谓Basic Block基本块，是指程序一顺序执行的语句序列，其中只有一个入口和一个出口，入口就是其中的第—个语句，出口就是其中的最后一个语句。对一个基本块来说，执行时只从其入口进入，从其出口退出。一个基本块的出口到另一个基本块的入口被称为edge。

​		基本块频率（Basic Block frequency）是估计不同基本块的相对频率的度量指标。块频率与入口块频率的比率是块对函数的每个入口执行的期望次数。

### **分支概率**

​		分支概率（[Branch Probabilit](https://llvm.org/docs/BlockFrequencyTerminology.html#id2)）被称为具有多个后继块具有与每个outgoing edge关联的概率，对于给定的块，它的outgoing分支概率的和应该是1.0。

### **分支权重**

​		分支权重（[Branch Weight](https://llvm.org/docs/BlockFrequencyTerminology.html#id3)）是以整数形式表示，而不是分数。一个给定的edge的分支概率是它自己的权值除以前驱outgoing  edge的权值之和。	

```
For example, consider this IR:

define void @foo() {    
	; ...    
	A:        
		br i1 %cond, label %B, label %C, !prof !0    
	; ... 
} !0 = metadata !{metadata !"branch_weights", i32 7, i32 8}
```

简单的图表示：
```
A -> B  (edge-weight: 7)
A -> C  (edge-weight: 8)
```

从A块到B块的概率是7/15，从A块到C块的概率是8/15。有关分支权重IR表示的详细信息，请参阅[LLVM Branch Weight Metadata](https://llvm.org/docs/BranchWeightMetadata.html)。

### **Implementation: a series of DAGs**

​		块频率计算的实现分析了每个循环，自下而上，忽略了backedges； 即作为DAG。 处理完每个循环后，将其打包以充当其父循环（或函数）DAG分析中的伪节点。

### **Block Mass**

​		对于每个DAG，入口节点被分配 a mass of UINT64_MAX，mass根据分支权重分配给后继节点。Block Mass 使用定点表示，其中UINT64_MAX表示1.0，0表示略高于0.0的数字。在mass  完全分布后，在将出口节点与入口节点分隔开的DAG的任意一条分割线上，一条切边后的节点块质量之和应等于UINT64_MAX。换句话说，mass 通过DAG是守恒的。如果一个函数的基本块图是DAG，那么block masses 就是有效的块频率。这在实践中效果很差，因为downstream 用户依赖于将块频率加在一起而没有达到最大值。

### **Loop Scale**

​		Loop scale是一个指标，它指示一个循环迭代每个入口的次数。当mass 通过loop’s DAG 分布时，backedge mass 被收集。此backedge mass 用于计算出口频率，从而计算 loop scale。

### **Implementation: Getting from mass and scale to frequency**

​		在分析完整的DAG序列之后，每个块都有一个mass (如果有的话，在其包含的循环中是局部的)，每个循环伪节点都有一个循环规模和它自己的mass (来自它的父节点的DAG)。可以通过将这些mass 和loop scales一起得到一个初始频率分配(入口频率为1.0)。给定的块频率是它的mass、包含循环伪节点的mass和包含loops’ loop scales。由于downstream 用户需要整数(而不是浮点数)，因此这个初始频率分配将根据需要转移到uint64_t范围内。

### **Block Bias**

Block bias 是用来指示在函数执行期间对给定块的偏差。其思想是偏差可以单独用于指示一个块是相对热还是相对冷，或者用于比较两个块来指示一个块比另一个块更热或更冷。

提出的计算涉及计算一个参考块的频率，其中:

- 每个分支的权值假设为1(即，每个分支概率分布为偶数)，

- 忽略loop scales

这个参考频率代表了无偏图中的块频率。偏差是块频率与参考块频率的比值。

