---
tags:
  - ml-compiler
  - compilation
  - IR
---
[[LLVM]] 的一个 sub-project，compiler infrastructure project。作为 LLVM 的一个子项目，自然也是[开源](https://github.com/llvm/llvm-project/tree/main/mlir/)的。

全称为 **M**ulti-**L**ayer IR，而并非 Machine-Learning IR。 

从名字也可以看出，MLIR 的设计本身是 general-purpose 的，并不专门面向 ml-compiler。

有一种说法是 MLIR 是“编译器的编译器”，更像一组用来打造 domain-specific compiler 的 toolkit。

已经成为 ml-compiler 领域的 infrastructure，MLIR 可以用来实现 ml-compiler，

有许多不同 dialect，分别对应不同的抽象 layer，具有自己的优化 pass。

MLIR 的编译是逐层下降（progressive lowering）的，即可能经过很多层 dialect 的转换，在 IR 的转换过程中逐渐添加对更底层信息的抽象。

最终会被 lower 到 [[LLVM IR]]。