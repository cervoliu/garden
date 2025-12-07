---
tags:
  - ml-compiler
  - tensor
---
开源: https://github.com/triton-lang/triton

Triton: a language and compiler for writing highly efficient custom Deep-Learning primitives. The aim of Triton is to provide an open-source environment to write fast code at higher productivity than CUDA, but also with higher flexibility than other existing DSLs.

曾经由 OpenAI 支持，目前 (22年之后) 由 Meta 主导开发（PyTorch 将 Triton 合并进 PyTorch 2.0 的 TorchInductor 后端）。

绕过 CUDA，在 python 层面直接写 pythonic-DSL 描述的 kernel。使用 MLIR 作为编译栈。编译流程大概是 python --> ttir (triton ir)--> ttgir (triton gpu ir) --> llvm ir。

相比于 CUDA 做了更多抽象，简化算子开发: 
比如图中所示，一些例如 shared memory，barriers 等抽象在 triton programming model 里对 programmer 是透明的。

![[Pasted image 20251204161637.png]]

具有与 [[CUDA Parallel Programming Model | CUDA 编程模型]] 完全不同的一套自己的 Triton Programming Model，有空再补吧。这套编程模型假定了目标硬件是某种类似 GPU 的架构，基本可以认为面向 nvidia gpu。

Meta 联合自己的 Pytorch 生态，使用 Triton 参与设计自研芯片 MTIA 的软件栈：

![[MTIA software stack.png]]