---
layout:     post
title:      "Transformer实践中的小例子"
subtitle:   "Transformer实践中的小例子"
date:       2025-02-12
author:     "ZBX"
header-img: "img/tag-bg.jpg"
tags:
    - LLM
---

## 词嵌入矩阵V

假设我们正在进行一个**机器翻译**任务，输入的是英文句子，输出是翻译成法文的句子。为了简单起见，假设我们使用了一个非常小的模型。

- **batch_size**：一次性处理的样本数量，假设是 2，表示我们有 2 个英文句子要翻译。

- **seq_len**：每个句子的最大长度。假设每个句子有 4 个词。

- **d_model**：每个词的表示维度。假设我们的词嵌入维度是 6，即每个词用一个 6 维向量表示。

### **1. 输入句子的表示**

假设我们有两个英文句子：

- "I am a student."
- "You are a teacher."

我们将这些句子转换成词嵌入向量。为了简单起见，我们假设词嵌入后的表示如下（每个词表示为一个 6 维向量）：

```python
import torch

# batch_size = 2, seq_len = 4, d_model = 6
V = torch.tensor([[[0.1, 0.2, 0.3, 0.4, 0.5, 0.6],  # "I"
                  [0.2, 0.3, 0.4, 0.5, 0.6, 0.7],  # "am"
                  [0.3, 0.4, 0.5, 0.6, 0.7, 0.8],  # "a"
                  [0.4, 0.5, 0.6, 0.7, 0.8, 0.9]],  # "student"
                 
                 [[0.9, 0.8, 0.7, 0.6, 0.5, 0.4],  # "You"
                  [0.8, 0.7, 0.6, 0.5, 0.4, 0.3],  # "are"
                  [0.7, 0.6, 0.5, 0.4, 0.3, 0.2],  # "a"
                  [0.6, 0.5, 0.4, 0.3, 0.2, 0.1]]]  # "teacher"

```

### 2. 张量的形状

`V` 的形状是 `(batch_size, seq_len, d_model)`，即：`V.shape == (2, 4, 6)`

### **3. 将 `V` 投影到多头注意力的头维度**

在多头注意力机制中，我们通常会将 `V` 拆分成多个头，每个头有一个较小的维度。

假设我们有 2 个注意力头 (`num_heads = 2`)，并且每个头的维度是 3（`head_dim = 3`），那么我们将 `V` 重新组织成：`(batch_size, seq_len, num_heads, head_dim)`

```python
# 拆分成 2 个头，每个头维度是 3
V = V.view(2, 4, 2, 3)  # (batch_size, seq_len, num_heads, head_dim)

# 交换 seq_len 和 num_heads 维度
V = V.transpose(1, 2)  # (batch_size, num_heads, seq_len, head_dim)

print(V.shape)  # 输出: torch.Size([2, 2, 4, 3])
```

这是表示2个句子，每个句子两个注意力头，每个句子4个单词，每个头的维度是3。

## trg[:, :-1]

`trg_input = trg[:, :-1]` 这行代码的意思是对 `trg` 张量进行切片，保留所有样本的所有行和从第一个元素到倒数第二个元素的所有列。

### **详细解析**

- `trg` 是一个二维或三维的张量，通常用于表示**目标序列**（比如翻译任务中的目标语言）。
- `:` 表示对所有行进行操作（即所有样本）。
- `:-1` 表示对每一行的所有列进行切片，从第一个元素到倒数第二个元素（不包含最后一个元素）。

### **例子：**

假设 `trg` 的形状为 `(batch_size, seq_len)`，其中 `batch_size` 是批次大小，`seq_len` 是序列的长度。

例如：

```python
python复制编辑import torch

# 假设 trg 是一个 (3, 5) 的张量，表示 3 个样本，每个样本有 5 个词（序列长度为 5）
trg = torch.tensor([
    [1, 2, 3, 4, 5],   # 第 1 个样本
    [6, 7, 8, 9, 10],  # 第 2 个样本
    [11, 12, 13, 14, 15]  # 第 3 个样本
])

# trg_input = trg[:, :-1] 会去掉每个样本的最后一个元素
trg_input = trg[:, :-1]

print(trg_input)
```

**输出：**

```
tensor([[ 1,  2,  3,  4],
        [ 6,  7,  8,  9],
        [11, 12, 13, 14]])
```

#### 解释：

- 原始的 `trg` 张量形状是 `(3, 5)`，表示 3 个样本，每个样本包含 5 个词。
- 切片 `trg[:, :-1]` 表示保留每个样本的前 4 个词（从第一个到倒数第二个），即去掉了每个样本的最后一个词。
- `trg_input` 的形状变成了 `(3, 4)`，表示每个样本的序列长度从 5 变成了 4。

------

### **常见用法**

在很多自然语言处理（NLP）任务中，比如机器翻译，`trg_input = trg[:, :-1]` 这种操作通常用来**去除目标序列的最后一个词**，因为通常在训练中，我们使用目标序列的前 n-1 个词作为输入，预测下一个词（即序列的第 n 个词）。这样，模型能够学习到根据之前的词预测下一个词的关系。

例如：

- **训练时**：使用目标序列的前 n-1 个词作为输入（`trg_input`），并用真实的目标序列作为目标输出（`trg`）。
- **推理时**：根据模型预测的结果作为输入来生成后续词。

## 源语言掩码&目标语言掩码

### **source mask**

`source mask`用于在 Transformer 模型的编码器中屏蔽填充符（`<pad>`）的位置，确保模型在计算注意力时不关注这些无效位置。

```
src_mask = (src != input_pad).unsqueeze(-2)
```

```
src = torch.tensor([[1, 2, 0, 0], [3, 4, 5, 0]])  # batch_size=2, seq_len=4
input_pad = 0
mask = (src != input_pad)
# mask = [[True, True, False, False], [True, True, True, False]]
mask = mask.unsqueeze(-2) # mask.shape = (2, 1, 4)
# mask = [
#    [[True, True, False, False]],   # 第一个句子的掩码
#    [[True, True, True, False]]     # 第二个句子的掩码
#]
```



### create mask

```python
def create_masks(src, trg):
    # 源语言掩码（忽略填充符）
    # src 是输入序列的张量，形状为 (batch_size, seq_len)，包含词索引
    # input_pad 是填充符 <pad> 在词表中的索引
    # src != input_pad 会生成一个布尔张量，标记哪些位置是有效词（非填充符），形状为 (batch_size, seq_len)
    src_mask = (src != input_pad).unsqueeze(-2)  # [batch_size, 1, seq_len]

    if trg is not None:
        # 目标语言掩码（填充符掩码 + 未来信息掩码）
        trg_pad_mask = (trg != target_pad).unsqueeze(-2)  # [batch_size, 1, seq_len]
        trg_len = trg.size(1)
        trg_sub_mask = torch.tril(  # 下三角掩码（允许自回归）
            torch.ones((trg_len, trg_len), device=trg.device)
        ).bool()
        trg_mask = trg_pad_mask & trg_sub_mask  # 合并两种掩码
    else:
        trg_mask = None

    return src_mask, trg_mask
```

输入例子：

```
# 假设：
input_pad = 0  # 源语言填充符索引
target_pad = 0 # 目标语言填充符索引
batch_size = 2
src_seq_len = 4
trg_seq_len = 3

# 样例输入：
src = torch.tensor([
    [1, 2, 3, 0],  # 第1个样本（末尾填充1个<pad>）
    [4, 5, 0, 0]   # 第2个样本（末尾填充2个<pad>）
]) # shape [2,4]

trg = torch.tensor([
    [6, 7, 8],     # 第1个目标序列（无填充）
    [9, 0, 0]      # 第2个目标序列（末尾填充2个<pad>）
]) # shape [2,3]

# 函数返回结果：
src_mask = [
    [[True, True, True, False]],  # 第1个样本掩码
    [[True, True, False, False]]  # 第2个样本掩码
] # shape [2,1,4]

trg_mask = [
    [[[ True, False, False],      # 第1个目标序列掩码
      [ True,  True, False],
      [ True,  True,  True]]],
    
    [[[ True, False, False],      # 第2个目标序列掩码（注意pad位置）
      [False, False, False],      # 第2行全False（因为trg[1,1]是<pad>）
      [False, False, False]]]
] # shape [2,1,3,3]

```

