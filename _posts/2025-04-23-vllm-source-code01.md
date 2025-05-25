---
layout:     post
title:      "vLLM source code study"
subtitle:   "vLLM source code study"
date:       2025-04-23
author:     "ZBX"
header-img: "img/tag-bg.jpg"
tags:
    - LLM
    - vLLM
---

## PagedAttention

#### 1. **核心类 `PagedAttention`**

```python
class PagedAttention(nn.Module):
    def __init__(self, num_heads, head_size, scale, num_kv_heads=None, sliding_window=None):
        super().__init__()
        self.num_heads = num_heads
        self.head_size = head_size
        self.scale = float(scale)
        self.num_kv_heads = num_heads if num_kv_heads is None else num_kv_heads
        self.sliding_window = sliding_window

        assert self.num_heads % self.num_kv_heads == 0
        self.head_mapping = torch.repeat_interleave(
            torch.arange(self.num_kv_heads, dtype=torch.int32, device="cuda"),
            self.num_queries_per_kv
        )

        if self.head_size not in _SUPPORTED_HEAD_SIZES:
            raise ValueError(f"head_size ({self.head_size}) is not supported.")
```

- **功能**：实现基于分页的多头注意力机制，支持提示（prompt）和生成（generation）阶段的高效计算。
- 关键参数：
  - `num_heads`: 查询（Query）头的数量。
  - `head_size`: 每个头的维度。
  - `scale`: 注意力缩放因子（通常为 `1/sqrt(head_size)`）。
  - `num_kv_heads`: Key-Value头的数量（默认与查询头相同）。
  - `sliding_window`: 局部注意力窗口大小（若启用，限制每个token只能关注前`sliding_window`个位置）。

- **关键初始化步骤**：
   - **头映射（head_mapping）**：当 `num_kv_heads < num_heads` 时，每个KV头对应多个查询头。例如，若 `num_heads=8`，`num_kv_heads=4`，则每个KV头对应2个查询头。通过 `repeat_interleave` 生成映射索引。
   - **支持的头尺寸**：仅支持特定 `head_size`（如64、128等），确保与硬件优化（如Tensor Cores）兼容。

#### 3. **方法 `set_attn_bias`**

```python
def set_attn_bias(self, input_metadata: InputMetadata, dtype: torch.dtype) -> None:
    if input_metadata.attn_bias:
        return
    prompt_lens = input_metadata.prompt_lens
    attn_bias = BlockDiagonalCausalMask.from_seqlens(prompt_lens)
    if self.sliding_window is not None:
        attn_bias = attn_bias.make_local_attention(self.sliding_window)
    input_metadata.attn_bias.append(attn_bias)
```

1. **作用**：
   - 为提示（prompt）tokens设置因果掩码（Causal Mask），确保每个token只能关注自身及之前的token。
   - 若启用 `sliding_window`，则应用局部注意力，限制关注窗口大小。
2. **关键点**：
   - `BlockDiagonalCausalMask`：将不同prompt的长度分块，避免跨prompt的注意力干扰。
   - `make_local_attention`：对每个块应用局部窗口限制。

#### 4. **方法 `multi_query_kv_attention`**

- **用途**：处理提示阶段的注意力计算（非缓存模式）。
- 步骤
  1. 若Key-Value头数少于查询头数，通过重复扩展Key和Value的维度。
  2. 使用 `xformers` 的 `memory_efficient_attention_forward` 计算注意力。
  3. 将结果写入输出张量。

#### 5. **方法 `single_query_cached_kv_attention`**

- **用途**：生成阶段的注意力计算（基于缓存）。
- 实现
  - 调用自定义C++扩展 `single_query_cached_kv_attention`。
  - 输入包括缓存的Key/Value张量（`key_cache` 和 `value_cache`）。
  - 使用 `block_tables` 和 `context_lens` 管理缓存块和上下文长度。

#### 6. **前向传播 `forward`**

- 流程
  1. 输入处理
     - 将输入的 `query`、`key`、`value` 张量重塑为三维（头维度）。
     - 预分配输出张量 `output`。
  2. 提示阶段
     - 若存在提示标记（`num_prompt_tokens > 0`），调用 `multi_query_kv_attention`。
  3. 缓存操作
     - 等待缓存事件（`cache_event`）完成。
     - 将当前Key/Value存入缓存（通过 `cache_ops.reshape_and_cache`）。
  4. 生成阶段
     - 若存在生成标记（`num_generation_tokens > 0`），调用 `single_query_cached_kv_attention`。
  5. **输出**：将结果展平为二维张量返回。

#### 7. **子类 `PagedAttentionWithRoPE`**

- **扩展功能**：添加旋转位置编码（RoPE）。
- 关键步骤
  - **初始化**：根据配置选择RoPE类型（如线性缩放或动态缩放）。
  - **前向处理**：在注意力计算前对 `query` 和 `key` 应用RoPE编码。
  - **依赖**：`RotaryEmbedding` 类处理位置编码的具体实现。

#### 8. **子类 `PagedAttentionWithALiBi`**

- **扩展功能**：使用ALiBi（绝对位置嵌入通过偏置）替代传统位置掩码。
- 关键步骤
  - **初始化**：存储ALiBi斜率参数（`slopes`）。
  - 掩码设置
    - 为每个提示生成ALiBi偏置张量（`LowerTriangularMaskWithTensorBias`）。
    - 偏置张量形状需对齐到8的倍数（优化Tensor Core计算）。
  - 注意力计算
    - 提示阶段逐个处理每个提示（因xformers不支持动态序列长度的自定义偏置）。
    - 生成阶段将ALiBi斜率传递给底层C++函数。

#### 9. **关键优化点**

- 内存效率
  - 使用 `xformers` 的高效注意力实现（减少显存占用）。
  - 缓存机制（`key_cache`/`value_cache`）复用历史KV向量，避免重复计算。
- 性能优化
  - 张量形状对齐到8的倍数（适配Tensor Core）。
  - 异步缓存操作（通过 `cache_event` 等待完成）。

#### 10. **设计目标**

- **适用场景**：支持长上下文生成（如LLM推理）。
- 优势
  - 动态处理多提示输入（不同长度）。
  - 生成阶段仅需计算单个标记，显著加速推理。
  - 支持RoPE和ALiBi等位置编码方案，灵活适配不同模型需求。

#### 11. **代码结构说明**

- 模块依赖
  - `xformers`：高效注意力计算库。
  - `vllm.attention_ops`：自定义C++扩展实现核心算子。
  - `cache_ops`：管理KV缓存的存储和复制。
- 输入元数据
  - `InputMetadata` 包含上下文长度、块表（block tables）、缓存标记等，用于协调分页和掩码生成。



## Source Code

```python
"""Multi-head attention."""
from typing import Any, Dict, List, Optional

import torch
import torch.nn as nn
from xformers import ops as xops
from xformers.ops.fmha.attn_bias import (BlockDiagonalCausalMask,
                                         LowerTriangularMaskWithTensorBias)

from vllm import attention_ops
from vllm import cache_ops
from vllm.model_executor.input_metadata import InputMetadata
from vllm.model_executor.layers.rotary_embedding import (
    DynamicNTKScalingRotaryEmbedding, LinearScalingRotaryEmbedding,
    RotaryEmbedding)

_SUPPORTED_HEAD_SIZES = [64, 80, 96, 112, 128, 256]


class PagedAttention(nn.Module):
    # pylint: disable=line-too-long
    """GPT-style multi-head PagedAttention.

    This class takes flattened 1D query, key, and value tensors as input. The
    input 1D tensors can either contain prompt tokens or generation tokens, in
    addition to paddings.

    If the input tensors contain prompt tokens, the layout is as follows:

    |<---------------------- num_valid_tokens ---------------------->|
    |<--------------- num_prompt_tokens -------------->|
    |<--prompt_0-->|<--prompt_1-->|...|<--prompt_N-1-->|<--padding-->|

    Otherwise, the layout is as follows:

    |<------------------ num_valid_tokens ------------------->|
    |<------- num_generation_tokens (M) ------->|
    |<--generation_0-->|...|<--generation_M-1-->|<--padding-->|

    The prompts might have different lengths, while the generation tokens always
    have length 1. The paddings are appended to make the input length a multiple
    of 8, which is desirable for Tensor Cores.

    The class does the following:
    1. Perform multi_query_kv_attention for the prompts. This operation does
        not use the KV cache.
    2. Wait for the cache operations (e.g., swap, copy) to finish. The cache
        operations are issued by the cache engine before executing the forward
        pass of the model, and they are executed asynchronously.
    3. Reshape and store the input key and value tensors in the KV cache.
    4. Perform single_query_cached_kv_attention for the generation tokens.
        This operation reads the previous key and value tensors from the KV
        cache.
    5. Output a flattened 1D tensor.
    """

    def __init__(self,
                 num_heads: int,
                 head_size: int,
                 scale: float,
                 num_kv_heads: Optional[int] = None,
                 sliding_window: Optional[int] = None) -> None:
        super().__init__()
        self.num_heads = num_heads
        self.head_size = head_size
        self.scale = float(scale)
        self.num_kv_heads = num_heads if num_kv_heads is None else num_kv_heads
        self.sliding_window = sliding_window

        assert self.num_heads % self.num_kv_heads == 0
        self.num_queries_per_kv = self.num_heads // self.num_kv_heads
        self.head_mapping = torch.repeat_interleave(
            torch.arange(self.num_kv_heads, dtype=torch.int32, device="cuda"),
            self.num_queries_per_kv)

        if self.head_size not in _SUPPORTED_HEAD_SIZES:
            raise ValueError(f"head_size ({self.head_size}) is not supported. "
                             f"Supported head sizes: {_SUPPORTED_HEAD_SIZES}.")

    def set_attn_bias(
        self,
        input_metadata: InputMetadata,
        dtype: torch.dtype,
    ) -> None:
        del dtype  # Unused.
        if input_metadata.attn_bias:
            # Already set by a previous layer.
            return
        prompt_lens = input_metadata.prompt_lens
        attn_bias = BlockDiagonalCausalMask.from_seqlens(prompt_lens)
        if self.sliding_window is not None:
            attn_bias = attn_bias.make_local_attention(self.sliding_window)
        input_metadata.attn_bias.append(attn_bias)

    def multi_query_kv_attention(
        self,
        output: torch.Tensor,
        query: torch.Tensor,
        key: torch.Tensor,
        value: torch.Tensor,
        input_metadata: InputMetadata,
    ) -> torch.Tensor:
        """Normal attention for the prompt tokens.

        Args:
            output: shape = [num_prompt_tokens, num_heads, head_size]
            query: shape = [num_prompt_tokens, num_heads, head_size]
            key: shape = [num_prompt_tokens, num_kv_heads, head_size]
            value: shape = [num_prompt_tokens, num_kv_heads, head_size]
            input_metadata: metadata for paged attention.
        """

        if self.num_kv_heads != self.num_heads:
            # Project the key and value tensors to the desired number of heads.
            key = torch.repeat_interleave(key, self.num_queries_per_kv, dim=1)
            value = torch.repeat_interleave(value,
                                            self.num_queries_per_kv,
                                            dim=1)

        # TODO(woosuk): The unsqueeze op may incur some CPU overhead. Optimize.
        out = xops.memory_efficient_attention_forward(
            query.unsqueeze(0),
            key.unsqueeze(0),
            value.unsqueeze(0),
            attn_bias=input_metadata.attn_bias[0],
            p=0.0,
            scale=self.scale,
        )
        # TODO(woosuk): Unnecessary copy. Optimize.
        output.copy_(out.squeeze(0))
        return output

    def single_query_cached_kv_attention(
        self,
        output: torch.Tensor,
        query: torch.Tensor,
        key_cache: torch.Tensor,
        value_cache: torch.Tensor,
        input_metadata: InputMetadata,
    ) -> None:
        """PagedAttention for the generation tokens.

        Args:
            output: shape = [num_generation_tokens, num_heads, head_size]
            query: shape = [num_generation_tokens, num_heads, head_size]
            key_cache: shape = [num_blocks, num_kv_heads, head_size/x,
                block_size, x]
            value_cache: shape = [num_blocks, num_kv_heads, head_size,
                block_size]
            input_metadata: metadata for paged attention.
        """
        block_size = value_cache.shape[3]
        attention_ops.single_query_cached_kv_attention(
            output,
            query,
            key_cache,
            value_cache,
            self.head_mapping,
            self.scale,
            input_metadata.block_tables,
            input_metadata.context_lens,
            block_size,
            input_metadata.max_context_len,
            None,  # alibi_slopes
        )

    def forward(
        self,
        query: torch.Tensor,
        key: torch.Tensor,
        value: torch.Tensor,
        key_cache: Optional[torch.Tensor],
        value_cache: Optional[torch.Tensor],
        input_metadata: InputMetadata,
        cache_event: Optional[torch.cuda.Event],
    ) -> torch.Tensor:
        """PagedAttention forward pass.

        NOTE: The query, key, and value tensors must be sliced from a qkv
        tensor of shape [num_tokens, 3 * num_heads * head_size].

        Args:
            query: shape = [num_tokens, num_heads * head_size]
            key: shape = [num_tokens, num_kv_heads * head_size]
            value: shape = [num_tokens, num_kv_heads * head_size]
            key_cache: shape = [num_blocks, num_kv_heads, head_size/x,
                block_size, x]
            value_cache: shape = [num_blocks, num_kv_heads, head_size,
                block_size]
            input_metadata: metadata for paged attention.
            cache_event: event to wait for the cache operations to finish.

        Returns:
            shape = [num_tokens, num_heads * head_size]
        """

        # Reshape the query, key, and value tensors.
        query = query.view(-1, self.num_heads, self.head_size)
        key = key.view(-1, self.num_kv_heads, self.head_size)
        value = value.view(-1, self.num_kv_heads, self.head_size)

        # Pre-allocate the output tensor.
        output = torch.empty_like(query)

        # Compute the attention op for prompts.
        num_prompt_tokens = input_metadata.num_prompt_tokens
        if num_prompt_tokens > 0:
            # Prompt run.
            assert input_metadata.num_generation_tokens == 0
            self.set_attn_bias(input_metadata, dtype=query.dtype)
            self.multi_query_kv_attention(
                output[:num_prompt_tokens],
                query[:num_prompt_tokens],
                key[:num_prompt_tokens],
                value[:num_prompt_tokens],
                input_metadata,
            )

        # Wait until the cache op is done.
        if cache_event is not None:
            cache_event.wait()

        # Reshape the keys and values and store them in the cache.
        # When key_cache and value_cache are not provided, the new key
        # and value vectors will not be cached.
        num_valid_tokens = input_metadata.num_valid_tokens
        if (num_valid_tokens > 0 and key_cache is not None
                and value_cache is not None):
            # The stride is 3 because the key and value are sliced from qkv.
            key_to_cache = key[:num_valid_tokens]
            value_to_cache = value[:num_valid_tokens]
            slot_mapping = input_metadata.slot_mapping
            if input_metadata.to_cache is not None:
                key_to_cache = key_to_cache[input_metadata.to_cache]
                value_to_cache = value_to_cache[input_metadata.to_cache]
                slot_mapping = slot_mapping[input_metadata.to_cache]

            cache_ops.reshape_and_cache(
                key_to_cache,
                value_to_cache,
                key_cache,
                value_cache,
                slot_mapping,
            )

        if input_metadata.num_generation_tokens > 0:
            # Decoding run.
            assert input_metadata.num_prompt_tokens == 0
            assert key_cache is not None and value_cache is not None, (
                "key_cache and value_cache must be provided when "
                "generating tokens.")
            # Compute the attention op for generation tokens.
            self.single_query_cached_kv_attention(
                output[num_prompt_tokens:num_valid_tokens],
                query[num_prompt_tokens:num_valid_tokens], key_cache,
                value_cache, input_metadata)

        # Reshape the output tensor.
        # NOTE(woosuk): The output tensor may include paddings.
        return output.view(-1, self.num_heads * self.head_size)


class PagedAttentionWithRoPE(PagedAttention):
    """PagedAttention with rotary positional embedding."""

    def __init__(
        self,
        num_heads: int,
        head_size: int,
        scale: float,
        rotary_dim: int,
        max_position: int = 8192,
        base: int = 10000,
        num_kv_heads: Optional[int] = None,
        is_neox_style: bool = True,
        rope_scaling: Optional[Dict[str, Any]] = None,
        sliding_window: Optional[int] = None,
    ) -> None:
        super().__init__(num_heads,
                         head_size,
                         scale,
                         num_kv_heads,
                         sliding_window=sliding_window)
        if rope_scaling is None:
            self.rotary_emb = RotaryEmbedding(head_size, rotary_dim,
                                              max_position, base,
                                              is_neox_style)
        else:
            scaling_type = rope_scaling["type"]
            scaling_factor = rope_scaling["factor"]
            if scaling_type == "linear":
                self.rotary_emb = LinearScalingRotaryEmbedding(
                    head_size, rotary_dim, max_position, base, is_neox_style,
                    scaling_factor)
            elif scaling_type == "dynamic":
                self.rotary_emb = DynamicNTKScalingRotaryEmbedding(
                    head_size, rotary_dim, max_position, base, is_neox_style,
                    scaling_factor)
            else:
                raise ValueError(f"Unknown RoPE scaling type {scaling_type}")

    def forward(
        self,
        positions: torch.Tensor,
        query: torch.Tensor,
        key: torch.Tensor,
        value: torch.Tensor,
        key_cache: torch.Tensor,
        value_cache: torch.Tensor,
        input_metadata: InputMetadata,
        cache_event: Optional[torch.cuda.Event],
    ) -> torch.Tensor:
        """ PagedAttention forward pass with rotary embedding.

        Args:
            positions: shape = [num_tokens]
            query: shape = [num_tokens, num_heads * head_size]
            key: shape = [num_tokens, num_kv_heads * head_size]
            value: shape = [num_tokens, num_kv_heads * head_size]
            key_cache: shape = [num_blocks, num_kv_heads, head_size/x,
                block_size, x]
            value_cache: shape = [num_blocks, num_kv_heads, head_size,
                block_size]
            input_metadata: metadata for paged attention.
            cache_event: event to wait for the cache operations to finish.

        Returns:
            shape = [num_tokens, num_heads * head_size]
        """

        # Apply rotary embedding to the query and key before passing them
        # to the attention op.
        query, key = self.rotary_emb(positions, query, key)
        return super().forward(
            query,
            key,
            value,
            key_cache,
            value_cache,
            input_metadata,
            cache_event,
        )


class PagedAttentionWithALiBi(PagedAttention):
    """PagedAttention with ALiBi attention bias."""

    def __init__(self,
                 num_heads: int,
                 head_size: int,
                 scale: float,
                 slopes: List[float],
                 num_kv_heads: Optional[int] = None) -> None:
        super().__init__(num_heads, head_size, scale, num_kv_heads)
        assert len(slopes) == num_heads

        slopes = torch.tensor(slopes, dtype=torch.float32)
        self.register_buffer("alibi_slopes", slopes, persistent=False)

    def set_attn_bias(self, input_metadata: InputMetadata,
                      dtype: torch.dtype) -> None:
        if input_metadata.attn_bias:
            # Already set by a previous layer.
            return
        # Generates ALiBi mask for each prompt.
        for prompt_len in input_metadata.prompt_lens:
            bias = torch.arange(prompt_len, dtype=dtype)
            # Note(zhuohan): HF uses
            #     `bias = bias[None, :].repeat(prompt_len, 1)`
            # here. We find that both biases give the same results, but
            # the bias below more accurately follows the original ALiBi
            # paper.
            bias = bias[None, :] - bias[:, None]
            bias = bias.to(self.alibi_slopes.device)

            # When using custom attention bias, xformers requires the bias to
            # be sliced from a tensor whose length is a multiple of 8.
            padded_len = (prompt_len + 7) // 8 * 8
            bias = torch.empty(
                1,  # batch_size
                self.num_heads,
                prompt_len,
                padded_len,
                device=self.alibi_slopes.device,
                dtype=dtype,
            )[:, :, :, :prompt_len].copy_(bias)
            bias.mul_(self.alibi_slopes[:, None, None])
            attn_bias = LowerTriangularMaskWithTensorBias(bias)
            input_metadata.attn_bias.append(attn_bias)

    def multi_query_kv_attention(
        self,
        output: torch.Tensor,
        query: torch.Tensor,
        key: torch.Tensor,
        value: torch.Tensor,
        input_metadata: InputMetadata,
    ) -> torch.Tensor:
        """Attention with ALiBi bias for the prompt tokens.

        Args:
            output: shape = [num_prompt_tokens, num_heads, head_size]
            query: shape = [num_prompt_tokens, num_heads, head_size]
            key: shape = [num_prompt_tokens, num_kv_heads, head_size]
            value: shape = [num_prompt_tokens, num_kv_heads, head_size]
            input_metadata: metadata for paged attention.
        """
        if self.num_kv_heads != self.num_heads:
            # Project the key and value tensors to the desired number of heads.
            key = torch.repeat_interleave(key, self.num_queries_per_kv, dim=1)
            value = torch.repeat_interleave(value,
                                            self.num_queries_per_kv,
                                            dim=1)

        # FIXME(woosuk): Because xformers does not support dynamic sequence
        # lengths with custom attention bias, we process each prompt one by
        # one. This is inefficient, especially when we have many short prompts.
        start = 0
        for i, prompt_len in enumerate(input_metadata.prompt_lens):
            end = start + prompt_len
            out = xops.memory_efficient_attention_forward(
                query[None, start:end],
                key[None, start:end],
                value[None, start:end],
                attn_bias=input_metadata.attn_bias[i],
                p=0.0,
                scale=self.scale,
            )
            # TODO(woosuk): Unnecessary copy. Optimize.
            output[start:end].copy_(out.squeeze(0))
            start += prompt_len
        return output

    def single_query_cached_kv_attention(
        self,
        output: torch.Tensor,
        query: torch.Tensor,
        key_cache: torch.Tensor,
        value_cache: torch.Tensor,
        input_metadata: InputMetadata,
    ) -> None:
        """PagedAttention with ALiBi bias for the generation tokens.

        Args:
            output: shape = [num_generation_tokens, num_heads, head_size]
            query: shape = [num_generation_tokens, num_heads, head_size]
            key_cache: shape = [num_blocks, num_kv_heads, head_size/x,
                block_size, x]
            value_cache: shape = [num_blocks, num_kv_heads, head_size,
                block_size]
            input_metadata: metadata for paged attention.
        """
        block_size = value_cache.shape[3]
        attention_ops.single_query_cached_kv_attention(
            output,
            query,
            key_cache,
            value_cache,
            self.head_mapping,
            self.scale,
            input_metadata.block_tables,
            input_metadata.context_lens,
            block_size,
            input_metadata.max_context_len,
            self.alibi_slopes,
        )

```

