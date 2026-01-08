---
draft: true
---
## Related Work

Reusing SMT solving results:
- KLEE
- Green: Reducing, Reusing and Recycling Constraints in Program Analysis
- GreenTrie
- Utopia: Reusing Solutions Modulo Theories
- Effective Use of SMT Solvers for Program Equivalence Checking Through Invariant-Sketching and Query-Decomposition. 
- [[Cache-a-lot]]: 针对 unsat core 的 variable substitution
- 稍微没那么相关的：Speeding up SMT Solving via Compiler Optimization


### Bounded Equivalence Checking tool

- VerifOx -- A Tool for Path-wise Symbolic Execution

Formal Semantic of x86-64 ISA:
- Stratified Synthesis: Automatically Learning the x86-64 Instruction Set
- [[A Complete Formal Semantics of x86-64 User-Level Instruction Set Architecture]] 

Program Equivalence Checking:
- Semantic Program Alignment for Equivalence Checking

Regression Verification:
- llreve: Automatic Regression Verification


### Benchmarks

EqBench (2021):
- **Content:** ~270 pairs of C and Java programs (147 equivalent, 125 non-equivalent).
- non-linear arithmetic, complex loop refactorings, array manipulations

EquiBench (2025):
- 2400 program pairs, including CUDA kernels, x86-64 assembly, standard C

OpenSSL / NaCl: ...

---
## Our work

当前工作的局限点：
- 只能做 Bounded Equivalence Checking 的 Regression Verification。基于路径的 cache 方案预先假设了这一点。

当前工作容易被质疑的弱点（如果我是审稿人，我会这样提问，并且无法正面回答）：
- 如何保证 x86-64/arm64 汇编指令的语义编码正确？在已经有现有工作专门做形式化的汇编指令的语义编码，为什么不直接采用它们？
- unsat core 这整个章节没有任何创新点。


新的 Title: A Reuse-Based Regression Verification Framework (for x86-64 Assembly)

---
## References

