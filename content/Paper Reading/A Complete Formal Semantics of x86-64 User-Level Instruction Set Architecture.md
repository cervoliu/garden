---
tags:
  - PLDI19
  - formal-semantics
  - x86
---
在 $\mathbb{K}$ framework 中提供了 the most complete and thoroughly tested formal semantics of x86-64。

- Omit the relaxed memory model of x86-64 and thus the concurrency-related operations
	- Modeling concurrency is a complex but relatively orthogonal problem in the presence of small-step operational semantics

简单介绍了 stratified synthesis 的原理：假设我们要确定某一条指令 IS 的形式语义。
- 首先，得有一个信任的，语义已被证明的子集 B。子集 B 最初可以用人工的方式确定，然后逐渐增长。
- 其次，还需要一组 test input 的集合 T（each input is a processor state configuration），以及对应的 test output。output 可以通过 executing IS on T 来得到。
- 用 Stoke synthesize 指令序列，只允许用 B 中的指令。要求 synthesis 得到的指令序列在 T 上的 output 与 IS 一致。背后的 idea 其实就是用 B 中的指令去模拟 IS 的语义。
- 重复上一步的 synthesis 过程。假设得到了两个不同的指令序列 S 和 S'。它们的语义理应保持一致。并且它们只包含 B 里面的指令，所以它们的语义是已知的，这个时候用 equivalence checker 去判定 S 与 S' 的语义等价性。如果不等价，将 SMT solver 返回的 model 翻译成 test input，加入 T 中。
- This process of synthesizing instruction sequence candidates and accepting or rejecting them based on equivalence checking with previous candidates, is repeated until a threshold is reached. 这里有一个疑问，就是如何确定 accept 还是 reject。因为不同的 instruction sequence 可以聚成等价类，你也不知道 IS 在哪个等价类里面。

Instructions not included:
1. System-level instructions. related to the operating system, protection levels, I/O, cache lines, and other supervisor instructions。对 system-level instruction 的建模是最麻烦的，这需要对不同的 arch 和 os 建模，并且与本文的目的（formalism of *user-level* instruction）正交
2. x87 & MMX instructions, consisting of legacy floating-point and vector operations, resp.
3. Concurrency-related operations, including atomic operations and fences
4. Cryptography instructions

---

§3 Formalization of x86-64 Semantics 讲怎么得到 $\mathbb{K}$ 中的 formalism

-  将 strata 工作中已有的形式化指令迁移到 $\mathbb{K}$ 中，构成初始信任指令集 B，大概占了总数 60%。
- 尝试用老一套 stratified synthesis，但总体上失败了。跑了两天只学到了 70 条新指令的语义。
- 剩下的 40% 是人工完成的，对照 Intel manual 翻译成 $\mathbb{K}$ rules。

![[program configuration.png]]
- §3.3 讲 program configuration，这个建模还是比较标准和常规的。
	- our memory layout is "flat", in which all available memory locations can be addressed, but we do have logical partitions of the memory into sections like code, data and stack.
- §3.4 讲 semantics of execution environment。在 $\mathbb{K}$ 中建模程序的执行：首先，将 program instructions 加载到内存，然后取指令，执行。译码没有被形式化。
- §3.5 讲 semantics of individual instructions。

---

§4 Validation of Semantics 讲怎么验证 §3 中语义的正确性

- §4.1 Co-Simulation against Hardware。$\mathbb{K}$ framework 只要定义好了一个语言，就有了开箱即用的 executor 等工具，它们用两款真实的 Intel CPU 对比运行。
- §4.2 Comparing with Stoke

---
§5 Applications
