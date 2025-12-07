---
tags:
  - DSL
  - HPC
  - tensor
---
TVM 的一个前端 DSL，用来写高性能算子。开源: [Domain-specific language designed to streamline the development of high-performance GPU/CPU/Accelerators kernels](https://github.com/tile-ai/tilelang)

Tile Language (**tile-lang**) is a concise domain-specific language designed to streamline the development of high-performance GPU/CPU kernels (e.g., GEMM, Dequant GEMM, FlashAttention, LinearAttention). By employing a Pythonic syntax with an underlying compiler infrastructure on top of [[TVM]], tile-lang allows developers to focus on productivity without sacrificing the low-level optimizations necessary for state-of-the-art performance.

[![](https://github.com/tile-ai/tilelang/raw/main/images/MatmulExample.png)](https://github.com/tile-ai/tilelang/blob/main/images/MatmulExample.png)

似乎是北大的团队搞的，deepseek v3.2 也用它来写算子，关注一下。据说比 [[Triton]] 暴露的硬件抽象更底层，提供 finer-grained control。

调度策略与算子数据流解耦。