---
draft: true
tags:
  - MLIR
  - compilation
---
整理 [[MLIR]] 相关工作：

- CGO'25 [DialEgg: Dialect-Agnostic MLIR Optimizer using Equality Saturation with Egglog](https://dl.acm.org/doi/10.1145/3696443.3708957)
	- [Github: DialEgg: MLIR + Equality Saturation](https://github.com/AzizZayed/DialEgg)
	- 针对几个 case 设计了 rewrite rules. “toy-ish” work.
- ATC'25 HEC: Equivalence Verification Checking for Code Transformation via Equality Saturation
- PLDI'25 First-Class Verification Dialects for MLIR
	- 阅读笔记：[[First-Class Verification Dialects for MLIR]]
	- 见 [slides](<file:////Users/cervol/Documents/talks/first-class verification dialects for mlir.pdf>)
- CAV'22 SMT-Based Translation Validation for Machine Learning Compiler
	- [Github: mlir-tv](https://github.com/aqjune/mlir-tv)
	- 只对 tensor, memref, linalg 三个 dialect 做了 validation
- ITP'24 Verifying Peephole Rewriting In SSA Compiler IRs
	- 针对 SSA IR（包括 MLIR）的小范围重写，提出一个通用形式化框架和证明基础，可机械证明 peephole rewrite 的正确性
- 一个 ongoing 的超级优化工作，目前还没有 pub
		- 见 [Automatically-generating-rewrite-patterns-in-MLIR.pdf](<file:////Users/cervol/Documents/talks/Automatically-generating-rewrite-patterns-in-MLIR.pdf>)
- (International Journal of Software Engineering and Knowledge Engineering) A Systematic Translation Validation Framework for MLIR-Based Compilers

一些基于测试的工作:
- 具体列表可以参考第一篇的 related work，这里简单列几篇
- ASE'25 Finding Bugs in MLIR Compiler Infrastructure via Lowering Space Exploration
	- 利用 MLIR 的多路径 lowering 特性，对不同 lowering paths 进行结果比对并找出不一致性
	- lowering equivalence property: all valid paths for the same MLIR program should produce semantically equivalent results
- OOPSLA'25 DESIL: Detecting Silent Bugs in MLIR Compiler Infrastructure
- ISSTA'24 Fuzzing MLIR Compiler Infrastructure via Operation Dependency Analysis
- 