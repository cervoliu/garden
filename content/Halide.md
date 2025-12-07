---
tags:
  - HPC
  - DSL
---
https://github.com/halide/Halide

[PLDI'13 Halide: A Language and Compiler for Optimizing Parallelism, Locality, and Recomputation in Image Processing Pipelines](http://people.csail.mit.edu/jrk/halide-pldi13.pdf)

一种用于 HPC 场景的 DSL

核心思想: **separation** of **computation (algorithm)**  and **scheduling**，即**计算与调度分离**，这个 idea 有着很深远的影响

algorithm: 计算的数学逻辑
scheduling: 在硬件上的优化执行 (tiling / vectorization / parallel / fuse)

这种思想在后续的 TVM 中也有所体现：

>  tvm github repo:  [Halide](https://github.com/halide/Halide): Part of TVM's TIR and arithmetic simplification module originates from Halide. We also learned and adapted some parts of the lowering pipeline from Halide.