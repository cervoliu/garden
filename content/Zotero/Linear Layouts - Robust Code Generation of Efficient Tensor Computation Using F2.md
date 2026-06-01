---
tags:
  - paper-reading
  - asplos26
  - tensor
  - layout
  - algebra
  - codegen
title: "Linear Layouts: Robust Code Generation of Efficient Tensor Computation Using F2"
---
## Background

### GPU Arch

自从 Volta 架构开始，NVGPU 引入 Independent Thread Scheduling，每个线程具有独立的寄存器和栈，相互独立地执行大多数常规指令。然而，一部分特殊指令，例如 `mma` 指令利用 tensor core 并行计算，它们需要以 warp 为粒度调度、执行。`wgmma` (warp group mma) 更进一步，在 4 个 warp 上同时执行 `mma` 运算。AMDGPU 也有类似指令，如 `mfma`。

> Note that these instructions require data to be distributed across threads and warps, or reside in shared memory or special memory units (e.g., Tensor Memory on Blackwell [36]) in special layouts to yield correct results.

