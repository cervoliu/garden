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

### Triton Language and Compiler

只需要了解 Triton dialect (`tt`) 被降级到 TritonGPU dialect (`ttgpu`) 的过程中，所有 tensor 变量的类型都会附带上 layout 属性，用以表示硬件相关的访存机制即可。
### Linear Algebra Preliminaries

概念自查：向量空间，子空间，线性组合，线性无关，张成，线性基。

### $\mathbb{F}_{2}$ Mathematics

$\mathbb{F}_{2}$ 其实就是只有 $\{0,1\}$ 组成的有限域。$\mathbb{F}_{2}$ 上的加法被解释成逻辑 XOR，乘法被解释为逻辑 AND。

## Overview

![[layout manifest.png]]

![[Legacy Layouts.png]]

原始的 Triton Layout 体系中有两种 Layout：Memory Layout 和 Distributed Layout。前者用于在特殊的可编程内存区域（如 shared memory, tensor memory）上表示张量布局，后者用于表示“分散”在各个线程内部存储（寄存器）的张量布局。