---
title: " Cache-a-lot: Pushing the Limits of Unsatisfiable Core Reuse in SMT-Based Program Analysis"
tags:
  - SMT
  - arxiv
---
过去的 SMT unsat core reuse 方法只考虑了 canonicalization（注：原文用的是 canonization 这个词，查了一下这个词应该是宗教里面用来表示封圣之类的意思，计算机科学中的“标准化”应该用前者），而并没有考虑 variable substitution。举例来说，对于以下的三个公式：

A: `(x > y) /\ (y > z) /\ (z > x)`

B: `(b > c) /\ (c > d) /\ (d > b) /\ (a > b)`

C: `(a > b) /\ (b > c) /\ (c > d) /\ (d > b)`

显然 A, B, C 都是 unsat 的。给定 A 的 unsat 结果，基于 canonicalization 的 cache 方法能够识别 B，但不能识别 C，原因就在于 canonicalization 虽然会对变量做重命名，但其重命名是简单地按照变量出现的位置来的。

对 A 做 canonicalization 会生成类似这样的一个变量替换 $\{ x \mapsto v_1, y \mapsto v_2, z \mapsto v_3 \}$

对 B 做 canonicalization 会生成 $\{b \mapsto v_1, c \mapsto v_2, d \mapsto v_3, a \mapsto v_4\}$

这样可以发现 A subsumes B，但 C 相较于 B 只做了子句位置上的替换，上述方法就失效了。

本文的解决的核心问题是，给定一个 unsat core A，和一个任意的合取式 F，高效判定是否存在一组变量替换 $\sigma$ 使得 A subsumes F，从而直接判定 F 是 unsat 的，减少 solver invocation。当然这里的一个前提是，A 与 F 的一个子集首先在结构上是 match 的（论文称这样的 A 为 candidates），不然变量替换也替换个寂寞。那么保证在结构上 match 这个前提是通过 formula hash 时隐去 variable name 的信息 + bloom filter 高效过滤来完成的（Candidate Selection）。完成前提后，这个核心问题就叫 Candidate Testing。

解决方案是，首先建立起 clause 到 clause 两两之间的 unifying substitution（即替换完两遍一模一样）。这会形成一个表格，表格行和列分别是 A 和 F 的项数，内容是 unifying substitution。然后要从这个表格中判定，这些 unifying substitution 能否组成一个 complete substitution $\sigma$，使得 $\sigma(A) \subseteq F$。这个判定问题可以巧妙地转化为以 unifying substitution 构造的 table 上的 natural join 是否不为空。然后由于只用判定是否为空，不用完整求出 natural join，可以针对性地做一些优化。

实验：与 sota 的 utopia solver 对比。benchmarks 从 klee 和 kex 两个 symbolic executor 构造。对比了 speed, canonicalization impact, optimization impact。