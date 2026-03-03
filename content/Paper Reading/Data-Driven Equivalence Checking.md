---
tags:
  - OOPSLA13
  - equivalence-checking
  - paper-reading
---
本篇 DDEC 为 STOKE 项目发表的第二篇 paper，也是该项目第一篇等价性验证的 paper，是 [[Semantic Program Alignment for Equivalence Checking]] 的前身。发表于 OOPSLA'13。

> First, DDEC guesses a *simulation relation* [25]. Roughly speaking, a simulation relation breaks two loops into a set of pairs of loop-free code fragments. Logical formulas associated with each pair describe the relationship of the input states of the fragments to the output states of the fragments. Second, DDEC generates verification conditions encoding the x86 instructions contained in each loop-free fragment as SMT [7] constraints. Finally, DDEC constructs queries which verify that the guessed relationships between the code fragments in fact hold. By construction, if the queries succeed they constitute an inductive proof of equivalence of the two loops.

这片文章使用了 simulation relation 的叫法，而不是 alignment。

> Theorem 1. If all VCs are proven then the target is equivalent to the rewrite and the cutset and the invariants together constitute a simulation relation.

simulation relation 可以认为是一个成功完成证明的 alignment。虽然叫法不同，但本质上是相同的技术路线，即通过寻找合适的 cutpoints + invariants 完成等价性的归纳证明。

> We use the well-known concept of a *cutpoint* [39] to decompose equivalence checking of two loops into manageable sub-parts. A cutpoint is a pair of program points, one in each program. Cutpoints are chosen to divide the loops into loop-free segments.

