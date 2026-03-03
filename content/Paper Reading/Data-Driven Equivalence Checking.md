---
tags:
  - OOPSLA13
  - equivalence-checking
  - paper-reading
---
本篇 DDEC 为 STOKE 项目发表的第二篇 paper，也是该项目第一篇等价性验证的 paper，是 [[Semantic Program Alignment for Equivalence Checking]] 的前身。发表于 OOPSLA'13。

> First, DDEC guesses a *simulation relation* [25]. Roughly speaking, a simulation relation breaks two loops into a set of pairs of loop-free code fragments. Logical formulas associated with each pair describe the relationship of the input states of the fragments to the output states of the fragments. Second, DDEC generates verification conditions encoding the x86 instructions contained in each loop-free fragment as SMT [7] constraints. Finally, DDEC constructs queries which verify that the guessed relationships between the code fragments in fact hold. By construction, if the queries succeed they constitute an inductive proof of equivalence of the two loops.

本篇

> We use the well-known concept of a *cutpoint* [39] to decompose equivalence checking of two loops into manageable sub-parts. A cutpoint is a pair of program points, one in each program. Cutpoints are chosen to divide the loops into loop-free segments.

