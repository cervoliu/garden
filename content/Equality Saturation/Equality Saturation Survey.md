---
aliases:
  - 等式饱和技术调研
tags:
  - e-graph
  - equality-saturation
  - term-rewrite
---
POPL'21 egg: Fast and extensible equality saturation:
- efficient rebuilding，本质上可以认为是算法与数据结构中的一种经典优化: lazy approach。lazy-style rebuilding 是如何保证 e-matching 不会“漏”的？
- abstract interpretation on e-classes
OOPSLA'21 Rewrite rule inference using equality saturation
PLDI'23 Better Together: Unifying Datalog and Equality Saturation
- egg 的相同团队做的后续工作。
POPL'24 Guided Equality Saturation
- Case Study 了 RISE 上的 rewrite
OOPSLA'24 Fast and Optimal Extraction for Sparse Equality Graphs
OOPSLA'24 PolyJuice: Detecting Mis-compilation Bugs in Tensor Compilers with Equality Saturation Based Rewriting
ASPLOS'25 SmoothE: Differentiable E-Graph Extraction:
- support for differentiable cost model (in extraction)
POPL'25 Dis/Equality Graphs
- formalism for e-graphs
PLDI'25 Slotted E-Graphs: First-Class Support for (Bound) Variables in E-Graphs