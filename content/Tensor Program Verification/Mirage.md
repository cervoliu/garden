---
title: "Mirage: A Multi-Level Superoptimizer for Tensor Programs"
tags:
  - tensor
  - OSDI25
  - superoptimization
---
# Intro

Existing work on automatically optimizing tensor programs fall into two categories:
1. optimize the _schedule_ of a tensor program while fixing the algorithm, i.e. "how to compute"
	- Halide, TVM, and Ansor
2. optimize via _algebraic transformations_, i.e. "what to compute"
	- TASO, Grappler, Tensat, PET
	- operator substitution / fusion / reorganization

Mirage: multi-level superoptimizer that automatically search for optimization on 3 levels:
- algebraic transformations
- schedule transformations
- discovery of new custom kernels

# Multi-Level Graph Representation: µGraph

µGraph:
- Specify **the execution of a tensor program on GPUs**.
- contains hierarchical graphs at multiple levels to represent computation at the **kernel, block and thread levels.**
