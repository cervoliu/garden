---
draft: true
tags:
  - MLIR
  - compilation
---
整理 [[MLIR]] 相关工作：

- CGO'25 [DialEgg: Dialect-Agnostic MLIR Optimizer using Equality Saturation with Egglog](https://dl.acm.org/doi/10.1145/3696443.3708957)
	- [Github: DialEgg: MLIR + Equality Saturation](https://github.com/AzizZayed/DialEgg)
- ATC'25 HEC: Equivalence Verification Checking for Code Transformation via Equality Saturation
- PLDI'25 First-Class Verification Dialects for MLIR
- CAV'22 SMT-Based Translation Validation for Machine Learning Compiler
	- [Github: mlir-tv](https://github.com/aqjune/mlir-tv)
- (International Journal of Software Engineering and Knowledge Engineering) A Systematic Translation Validation Frameworkfor MLIR-Based Compilers

一些基于测试的工作:
- 具体列表可以参考第一篇的 related work，这里简单列几篇
- ASE'25 Finding Bugs in MLIR Compiler Infrastructure via Lowering Space Exploration
	- lowering equivalence property: all valid paths for the same MLIR program should produce semantically equivalent results
- OOPSLA'25 DESIL: Detecting Silent Bugs in MLIR Compiler Infrastructure
- ISSTA'24 Fuzzing MLIR Compiler Infrastructure via Operation Dependency Analysis