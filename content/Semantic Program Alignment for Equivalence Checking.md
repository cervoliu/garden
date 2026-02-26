---
tags:
  - equivalence-checking
  - PLDI19
---
> Program equivalence checking is commonly performed in two stages: the first stage is to construct a *product program* for the two programs by *aligning* them, and the second is proving a safety property, or *invariant*, of the resulting program [4, 40].

程序等价性的证明通常通过两步完成：第一步是通过**对齐**的方式构造关联起两个程序的**乘积程序**；第二步是在乘积程序上，用经典的归纳不变式的方法完成等价性的证明。
第二步沿用了关系程序验证中经典的构造关系不变式以验证关系属性的技术路线，在等价性验证的任务中本质上并无特殊之处，只是将程序等价性看成一种特殊的关系属性。因此，对于等价性验证这个更具体的任务，解决问题的关键在于第一步，即找出更好的程序对齐点（对齐谓词），从而构造出验证高效的乘积程序。

![[product-programs.png]]
上图展示了构造乘积程序的一个 motivating example。对于子图 (a)(b) 中的函数 f, g，(c) 给出了平凡的乘积构造，即简单的首尾拼接，然而 (c) 中的乘积程序 X 对于证明等价性没有提供任何帮助，因为分析 X 在本质上与分别对 f, g 分析并无二致，其依然需要对 f, g 中函数的完整概括（inductive loop invariant）。而 (d) 中的乘积程序 Y 则给出了更为精细的乘积构造，这种构造利用了 f, g 中的 while 循环迭代次数相同的语义性质（其实图中的循环条件 `*` 与循环体 `A`, `B` 并不能保证这一点，但论文就是这么写的，我们假设性质成立），从而将问题转化为证明一个相对更容易的关系不变式 $Inv$。

> Given two functions $f$ and $g$ along with test cases provided by the user, we build a *trace alignment*, which is a pairing of states in execution traces of f and g for each user-provided test case. Constructing the trace alignment is guided by the selection of a weak invariant, called an *alignment predicate*, that identifies pairs of machine states that should be aligned.

本文提出了一种基于语义的对齐。基于用户提供的输入测例，构造两个程序之间的 trace alignment。