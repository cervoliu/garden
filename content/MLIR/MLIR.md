---
tags:
  - ml-compiler
  - compilation
  - IR
  - MLIR
---

推荐阅读: 
- [MLIR 学习：Dialect，可扩展性的基石](https://mp.weixin.qq.com/s/BotO4zBCxTxR-Nf0TRO_KQ)
- [MLIR 学习：Operation，统一 IR 的原子](https://mp.weixin.qq.com/s/8uteeqmD0LmP9bcmBCkwXw)
- [MLIR 学习：IR Structure， 多层级语义的骨架](https://mp.weixin.qq.com/s/4xVzSvdVBYRVEhban6c-RQ)
- [MemRef Dialect：MLIR 中统一内存模型与优化的核心抽象](https://mp.weixin.qq.com/s/Yq6F_Tr7Vi5ENUwDtIZOIQ)
- [Transform Dialect：当 Pass 无法精准控制变换时](https://mp.weixin.qq.com/s/QLAHnNh_018x5r6TufNGFg)

[[LLVM]] 的一个 sub-project，compiler infrastructure project。作为 LLVM 的一个子项目，自然也是[开源](https://github.com/llvm/llvm-project/tree/main/mlir/)的。

全称为 **M**ulti-**L**ayer IR，而并非 Machine-Learning IR。 

从名字也可以看出，MLIR 的设计本身是 general-purpose 的，并不专门面向 ml-compiler。

有一种说法是 MLIR 是“编译器的编译器”，更像一组用来打造 domain-specific compiler 的 toolkit。

已经成为 ml-compiler 领域的 infrastructure，MLIR 可以用来实现 ml-compiler，

有许多不同 ==dialect===，分别对应不同的抽象 layer，具有自己的优化 pass。比较知名的 dialect 有 `linalg`（线性代数）、`scf`（结构化控制流）、`arith`（算术运算）、`memref`（内存引用）等。
我比较关心的 dialect 还有 `affine`（仿射）。

MLIR 的编译是逐层下降（progressive lowering）的，即可能经过很多层 dialect 的转换，在 IR 的转换过程中逐渐添加对更底层信息的抽象。

如果目标硬件是 GPU，最终会被 lower 到 [[LLVM IR]]（如果目标硬件是 NPU（例如 Ascend），似乎并不依赖 LLVM？）

![[MLIR dialect translation.png]]