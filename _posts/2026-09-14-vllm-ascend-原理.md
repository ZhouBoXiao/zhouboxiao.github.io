---
layout:     post
title:      "vLLM-Ascend 原理"
subtitle:   "vLLM 昇腾 NPU 硬件插件：从 PagedAttention 到插件机制"
date:       2026-09-14
author:     "ZBX"
header-img: "img/tag-bg.jpg"
tags:
    - LLM
    - vLLM
---

## vLLM-Ascend 是什么

vLLM-Ascend 是 vLLM 社区与华为昇腾团队联合维护的开源硬件插件，作用是把 vLLM 这套原生面向 CUDA 的推理引擎接到昇腾 NPU 上。它不 fork vLLM，也不改 vLLM 一行源码，只通过插件接口注入昇腾实现。用户侧装两个包即可运行：

```bash
pip install vllm vllm-ascend
```

启动时 vLLM 扫描插件入口，检测到 torch_npu 可用后激活 ascend 平台，日志里能看到：

```
INFO plugin ascend loaded.
INFO Platform plugin ascend is activated
```

此后 Transformer 类、MoE、Embedding、多模态模型都能跑在昇腾 NPU 上，调度、内存管理、采样全部复用 vLLM 主干。

## 一、整体分层：谁负责什么

| 层 | 职责 | 关键组件 |
| --- | --- | --- |
| 应用层 | OpenAI 兼容 API、离线推理 | API Server |
| vLLM 引擎核心 | 调度、内存管理、采样（设备无关） | Scheduler、PagedAttention |
| vllm-ascend 插件 | 设备接入的"翻译层" | Platform、Worker、ModelRunner、Attention 后端 |
| 昇腾软件栈 | NPU 算子与通信 | torch_npu、CANN（ATB）、HCCL |
| 昇腾硬件 | 算力与显存 | Atlas 800 · 昇腾 910B NPU |

![vLLM-Ascend 生态分层全景](../img/vllm-ascend-layers.svg)

> 自上而下：应用 → 引擎 → 插件 → 昇腾软件栈 → 硬件。紫色层即 vllm-ascend，本篇主角。

vLLM 只认设备无关接口，NPU 只认 CANN/torch_npu，插件层负责两者之间的翻译，三层可以独立升级互不侵入。

为什么做成插件而不是直接在 vLLM 主干里加后端？早期 vLLM 的各硬件后端各自持有 Executor、Worker、Runner、Attention 组件，非通用代码散落在主干里，加一个新后端要做侵入式修改，维护成本随后端数量增长。2024 年底社区通过 [Hardware Pluggable RFC](https://github.com/vllm-project/vllm/issues/11162)，把 `Platform` 模块做成插件入口，并重构了 `Executor`、`Worker`、`ModelRunner`、`AttentionBackend`、`Communicator` 五大组件的接口（[vLLM 官方博客](https://vllm-project.github.io/2025/05/12/hardware-plugin.html)）。vllm-ascend 是这套机制的首个落地验证项目，同批还有 IBM Spyre。

## 二、vLLM 引擎核心：为什么快

### 2.1 PagedAttention 分页 KV 缓存

LLM 是自回归模型，每生成一个 token，注意力层都要读取整个前缀的 K/V 向量，所以必须缓存，否则每步都要重算全部历史。KV 缓存的体积按下面的公式累加：

```
KV bytes / token = 2 × layers × kv_heads × head_dim × dtype_size
```

以 LLaMA-2-7B fp16 为例（32 层、32 个 KV 头、head_dim 128、无 GQA）：每个 token 约 512 KB，一条 4096 token 的序列就要 2 GB。几十条并发序列一累计，几十 GB 的 HBM 就没了，怎么管理这块显存直接决定吞吐上限。

传统做法给每条序列按 max_len 预留一整块连续显存，三个问题随之而来：序列提前结束，预留部分全部浪费（内部碎片）；不同序列的相同前缀（比如同一个 system prompt）无法共享显存；显存很快被"占坑"耗尽，并发数上不去。

![传统 KV 缓存与 PagedAttention 分页机制对比](../img/vllm-ascend-paged-attn.svg)

> 上：传统方式按 max_len 整段预留，虚线部分全部浪费；下：分页方式逻辑连续、物理离散，按需申请与归还块。

PagedAttention 把操作系统的虚拟内存分页搬了过来：KV 缓存被切成固定大小的块（vLLM 默认每块 16 个 token），物理块放在全局块池里，每条序列持有一张 Block Table 记录逻辑块到物理块的映射。序列逻辑上看到的还是一段连续缓存，物理上却分散在块池各处，用到哪块申请哪块，序列结束块即刻归还。这带来三重收益：

1. 碎片从"每条序列一大段"缩小到"每条序列尾块内一点"；
2. 相同前缀的序列可以直接指向同一批物理块，写时复制（copy-on-write）后再分叉，这就是前缀缓存（prefix caching）的底层支撑，V1 引擎默认开启；
3. 同样显存能容纳多得多的并发序列。

在 vllm-ascend 侧，块表里的物理块就是 910B 的 HBM，注意力算子必须支持按块表寻址读写 KV——这正是 Attention 后端要替换的核心原因。

### 2.2 连续批处理（Continuous Batching）

PagedAttention 解决"装得下"，连续批处理解决"不空转"。

静态批处理整批同进同出：批内 4 条序列长短不一，最短的 R1 早就生成完了，却必须占着卡位等到最长的 R4 结束，这段时间 NPU 在做无用功。连续批处理把调度粒度从"一批"细化到"一次迭代"：每生成一个 token 就重新组批，完成的序列立刻踢出、新请求立刻插入，配合 PagedAttention（新请求申请得到块就能入批），吞吐可以翻倍以上。

![静态批处理与连续批处理对比](../img/vllm-ascend-batching.svg)

> 上：静态批处理，短序列完成后只能空转等待整批结束；下：迭代级组批，相同时间窗完成 8 条请求。

### 2.3 Prefill：一次算完整个 prompt

连续批处理把请求分成两个计算特征截然相反的阶段，先看 Prefill。它要完成的事一句话概括：把一段 prompt 变成"可以开始生成的状态"——算出所有位置的 KV 存进缓存，并顺手用最后一个位置预测出第一个输出 token。

**按执行顺序拆开，一次 prefill 前向干了这些活**：

1. **组批**：Scheduler 从等待队列挑一批 prompt 拼成一次 prefill batch，总量受 token 预算约束（`max_num_batched_tokens`，如 8192），每条序列各带各的 Block Table；前缀缓存命中的请求先把共享前缀指向已有物理块，只计算新增后缀；
2. **Embedding 与位置编码**：token id 查表得到向量，按位置算 RoPE——只加在 Q、K 上，V 不带位置信息；
3. **逐层前向，每层四步**（LLaMA-7B 要把这套循环跑 32 遍）：
   - **QKV 投影**：所有位置一起做大 GEMM（三个投影通常融合成一个），每个位置各得一份 q、k、v。矩阵大而规整，是 prefill 算力消耗的大头；
   - **因果自注意力**：位置 i 的 q 与位置 0..i 的 k 做点积，softmax 后加权 v——三角掩码保证"只能往前看"。L 个位置同时进行，注意力矩阵理论上是 L×L，但 flash attention 类融合算子分块计算，从不把它物化到显存；GQA 模型下 kv 头少于 q 头，多个 q 头共享同一组 k/v；
   - **KV 写缓存**：本层的 k、v 按块表写入物理块，16 个 token 一块，写满申请新块——2.1 节那套分页机制在 prefill 侧的动作就是这里；
   - **FFN**：每个位置独立过 gate/up/down 投影，又一批大 GEMM；
4. **出口极小**：N 层跑完，只有最后一个位置的隐状态被投影成 logits、采样出第一个生成 token；其余位置的前向输出直接丢弃，价值全部沉淀在写进缓存的 KV 里。一整段的计算换来"1 个 token + 一份缓存"，这正是首 token 延迟（TTFT）的主要构成。

![Prefill 单次前向的内部流水](../img/vllm-ascend-prefill-pipeline.svg)

> 自上而下：Embedding 与 RoPE → 虚线框内逐层循环"QKV 投影 → 因果注意力（含三角掩码）→ KV 写块 → FFN" → 出口只取末位置。

为什么说 prefill 计算密集：全程由大 GEMM 和融合注意力主导，矩阵规模随 prompt 长度线性增长（注意力部分 L²），形状规整、无分支，910B 的 AI Core 可以持续满载。

### 2.4 Decode：一步一个 token 的循环

prefill 完成后，序列进入循环体：每步吃进上一步采样的 token，吐出下一个，直到 EOS 或长度上限。批里的 B 条序列各自循环，每步组成一个 batch 一起算。

**单步解码干了这些活**——与 prefill 同构，但处处缩小一号：

1. **输入**：上一步采样的 token id，每条序列各 1 个，整个输入张量是 B×1——prefill 里的 L 维坍缩成了 1；
2. **Embedding 与位置编码**：单 token 查表，RoPE 的位置号就是当前序列长度；
3. **逐层四步，全部缩小**：
   - **QKV 投影**：从大 GEMM 退化为矩阵乘向量（GEMV），单 token 的计算量小到喂不饱 AI Core；
   - **分页自注意力**：新位置的 q 与全部前缀 K/V 做注意力。K/V 不在本轮重算，而是按 Block Table 到块池里去读——本序列的物理块散落各处，算子逐块寻址；GQA 模型下只读 kv_heads 份。这是 AscendAttention 分页算子的主战场；
   - **KV 追加写**：本轮新算出的 k、v 写进序列最后一个物理块的空位，尾块写满就申请新块；
   - **FFN**：又是三个 GEMV；
4. **出口**：唯一位置的隐状态投影成 logits、采样出下一个 token——它就是下一轮的输入。

![Decode 单步的内部流水](../img/vllm-ascend-decode-pipeline.svg)

> 与 2.3 的 prefill 流水对照：投影退化为 GEMV；注意力从"算 L×L"变成"读全量前缀"；KV 从整段写入变成尾块追加。

每步计算量极小，要搬的东西却一样不少：权重全部过一遍，KV 缓存全部读一遍。每读一个字节只伴随很少的乘加，瓶颈从算力换成了 HBM 带宽，这是 decode 访存密集的真正含义。再叠加小算子问题：一步解码要下发几十个 kernel，每个都填不满，host 端调度开销占比陡增，NPU 相当一部分时间在"等指令"。

vllm-ascend 对此有三个对策。**并发 batch**：B 条序列的 decode 拼在一起算，权重读一次摊给所有序列。**ACL Graph**：把一步解码整图捕获后下沉 NPU 一次执行，host 只负责搬输入输出。**推测解码**：草稿模型一次生成多个候选 token，目标模型一次前向全部验证，把单 token 成本摊薄。

连续批处理的动作也发生在这里：每步迭代结束，完成的序列立刻踢出、新请求立刻插入，decode 的循环始终满载。

两个阶段的量级不对称是调度的根本张力：prefill 一出现就抢占整卡算力，若并发里混进一个长 prompt，它的一次 prefill 会让批里其他请求的 decode 集体停摆。chunked prefill 把长 prompt 切成小块分多次执行，把每次 prefill 的占用压到一两个迭代之内，保住批内 decode 的节奏——注意切块之后，后一块的注意力要读前一块已写入的 KV，prefill 从第二块起就带着"读缓存"的成分。

![Prefill 与 Decode 阶段计算特征对比](../img/vllm-ascend-prefill.svg)

> 上：Prefill 整段并行、KV 整段写入，计算密集；中：Decode 单 token 查询全部前缀、只写 1 块，访存密集；下：Chunked Prefill 切块逐次执行。

在 vllm-ascend 里，两个阶段走不同后端：prefill 贪图算子融合度（ATB 高融合算子），decode 贪图低调度开销（ACL Graph 整图 + 按块表寻址的分页注意力算子）。2.1 节强调注意力算子必须支持块表寻址，第一个答案就在 decode 这里——每步都要按 Block Table 去读散落的前缀 KV。

### 2.5 V1 引擎架构

当前默认的 V1 引擎按进程拆分：

```
API Server（OpenAI 兼容接口）
   │
AsyncLLM（前端，异步管理请求生命周期）
   │  ZMQ 消息队列
EngineCore 进程（Scheduler + KV Cache 管理器）
   │
Executor → Worker（ModelRunner → AttentionBackend）
```

Scheduler 跑在 CPU 上做组批决策，Worker 跑在 NPU 上执行模型，两者解耦。这个结构决定了插件要做的事：只替换"下锤子的手"（Worker 侧），不动"决定怎么锤的脑子"（Scheduler 侧）。

## 三、插件挂接机制：零侵入的五个替换点

注册链路分三步，全部通过 Python entry points 完成：

```python
# setup.py 中的注册声明
setup(
    entry_points={
        'vllm.platform_plugins': ["ascend = vllm_ascend:register"]
    }
)
```

1. **注册**：vllm-ascend 在 `vllm.platform_plugins` 组下登记 `ascend` 入口；
2. **发现**：vLLM 启动时扫描该组的所有入口，逐个加载插件包；
3. **激活**：`register()` 返回一个 `Platform` 类，探测逻辑发现 torch_npu 可用、设备是 NPU 时选中它，后续所有设备相关组件都从它这里拿工厂方法。

![vllm-ascend 插件挂接机制](../img/vllm-ascend-plugin.svg)

> 顶部为安装激活流程，下方为逐行「接口 → 昇腾实现」替换：接口在 vLLM 主干，实现在插件仓库。

替换点一共五个，接口在 vLLM 主干，实现在插件仓库：

| vLLM 抽象 | vllm-ascend 实现 | 职责 |
| --- | --- | --- |
| `Platform` | NPUPlatform | 设备探测、NPU 属性、初始化 |
| `WorkerBase` | NPUWorker | 昇腾卡上的执行循环 |
| `ModelRunnerBase` | NPUModelRunner | 批数据组装、构图、量化适配 |
| `AttentionBackend` | AscendAttention | 分页注意力算子（ATB） |
| `Communicator` | PyNPUCommunicator | HCCL 集合通信（TP/PP） |

调度器、PagedAttention 逻辑、采样器全部复用主干，插件只写"不同"，不重写"相同"。这就是插件架构的价值：vLLM 升级版本，vllm-ascend 只需跟进接口变化；昇腾出新芯片，vLLM 主干一行不动。

## 四、昇腾侧关键技术

**ATB 算子**。CANN 里的 Ascend Transformer Boost 加速库提供高融合度算子：flash attention、分页注意力、RMSNorm 等。AscendAttention 后端调用的就是它们，让 prefill/decode 都能按块表寻址读写 KV。

**图模式（ACL Graph / torchair）**。它解决 decode 的"小算子税"：一步解码几十个 kernel，每个都小到喂不饱 AI Core，而 host 每下发一个算子都要走一遍框架→runtime→驱动的固定开销。逐个下发时，NPU 大量时间花在等下一条指令上，空泡比计算还长。

原理分两个阶段。**捕获（capture）**：首次执行某个形状的解码步时，把整条算子序列连同依赖关系、内存地址记录成一张静态图；**重放（replay）**：之后每步只需把新数据写进固定地址的输入 buffer、一次调用整图——host 从"下发 N 次"变成"下发 1 次"，调度开销从 N 份 launch 降为常数。

图是静态的，decode 的动态性靠三个约定吸收：输入输出 tensor 的地址、形状固定，变化的只是内容；batch 维度按桶捕获（1、2、4、8……预设档位），运行时向上取整到最近的桶做 padding 执行；分页 KV 的块表以 tensor 输入的形式传入——地址固定、内容每步更新，这正是图模式与 PagedAttention 能共存的关键。代价是每个桶要额外存一套图和静态 buffer，含动态分支的算子无法入图、只能退回 eager。

在 vllm-ascend 里构图由 NPUModelRunner 负责：decode 走图模式，prefill 因为长度动态、捕获不划算而保持 eager。

![ACL Graph 图模式原理](../img/vllm-ascend-aclgraph.svg)

> 上：eager 模式逐算子下发，NPU 空泡夹在执行块之间；下：capture 记录整图、replay 一次下发连续执行；底部为静态图装下动态 decode 的三个约定。

**量化**。支持 W8A8/W8A16 权重量化与 KV Cache 量化，同等 HBM 容量下容纳更多并发。从源码编译 vllm-ascend 时自定义算子构建默认启用，需要先装 gcc、cmake 等工具链，不需要时可用环境变量 `COMPILE_CUSTOM_KERNELS=0` 关闭。

**分布式与调度**。TP/PP 并行走 HCCL；针对 DeepSeek 类模型有 TBO（two-batch overlap，双批重叠）掩盖通信开销；MoE 模型有 EPLB 专家负载均衡，缓解专家命中不均导致的卡间忙闲差。

**推测解码**。采用提议者-验证者架构：`vllm_ascend/spec_decode/` 负责生成草稿 token（从 n-gram 匹配到草稿模型），`vllm_ascend/sample/` 里的拒绝采样器根据目标模型分布验证接受，一次前向产出多个 token。

## 五、一个请求的端到端流程

把上面所有机制串成一条时间线：

1. **请求接入**：prompt 经 OpenAI 兼容 API 进入，分词为 token 序列，进等待队列；
2. **调度组批**：Scheduler 按迭代级组批，完成的序列出、新请求进；前缀缓存命中的请求直接复用已有物理块；
3. **Prefill**：一次算完整个 prompt，KV 写入物理块池（长 prompt 会被切块）；
4. **Decode**：每步生成 1 个 token，ACL Graph 整图下沉 NPU 执行，KV 追加写入尾块；
5. **输出**：logits 采样、detokenize，流式返回给客户端。

![请求在 vllm-ascend 上的端到端流程](../img/vllm-ascend-journey.svg)

> 紫色节点是 NPU 计算热点，底部横条为插件替换的执行内核。

多卡场景下，模型按 TP 切分到各 NPU，每层前向后通过 HCCL 做 all-reduce。

## 六、学习路径

按四阶段推进，每阶段都以上一阶段为地基：

1. **跑通体感**：在昇腾环境安装 vllm + vllm-ascend，跑一个 Qwen 小模型，对照启动日志里的 "Platform plugin ascend is activated"，把分层和替换点在日志里逐一对上号；
2. **夯实 vLLM 原理**：精读 [PagedAttention 论文](https://arxiv.org/abs/2309.06180)（vLLM: Easy, Fast, and Cheap Serving）和 vLLM 官方文档的 V1 Engine、Scheduler 章节，吃透"为什么快"；
3. **精读插件源码**：按调用顺序读 [vllm-ascend 仓库](https://github.com/vllm-project/vllm-ascend)：`setup.py` 的 entry point → `vllm_ascend/platform.py` → worker → attention backend，对照第五节的替换点表格；
4. **深入昇腾栈**：CANN / ATB / torch_npu 文档，理解算子层如何支撑上层抽象。

一句话总结：vLLM 管"聪明地调度"，昇腾管"暴力地计算"，vllm-ascend 用五个标准接口把两者焊在一起，互不侵入、各自演进。完整文档见 [vLLM Ascend 官方文档](https://docs.vllm.ai/projects/ascend/en/latest/)。
