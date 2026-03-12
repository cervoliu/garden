---
tags:
  - OOPSLA18
  - relational-verification
  - paper-reading
---
![[semantic conflict freedom.png|627]]

针对软件工程中的 3 路合并，定义了语义冲突的问题，并将语义无冲突（semantic conflict-freedom）形式化为参与 3 路合并的 4 个程序版本（构成 4 元组，称为 3 路合并场景）之间的一种关系属性（relational safety property），从而将识别合并语义冲突转化为关系验证问题。

其中 $out$ 表示可观测输出状态（observable output）

具体的验证流程包括**关系后置条件（relational post-condition）的生成**以及**乘积程序的构造**。

首先，为了复用程序版本之间的相似语法结构，以实现「组合式的推理 (compositional reasoning)」，第一步是将多个程序版本表示为  hole_program + program_edits 的形式。具体来说，hole_program 表示一个含有空洞（holes）的程序模板，它对应了多个程序版本共享的语法结构。每个具体的程序版本都有自己对应的 program_edits 列表，满足将列表中的程序片段填入 hole_program 就能得到完整的程序。

![[RPC inference.png|493]]
规则 (3, 4)：关系后置条件的生成，对于顺序结构和分支结构来说是平凡的。

规则 (2)：对于 hole_program 中被空洞隔开的共享程序片段，本文采用了上近似的抽象以减轻验证负担，具体来说，对于共享片段，用依赖分析求出该片段依赖的变量名集合 $\vec x$，将不同片段的关系后置条件用不同的未解释函数符号 $F(\vec x)$ 抽象。

> The RPC computation engine from Section 5.1 requires an inductive loop invariant relating variables from the four program versions. Our implementation automatically infers relational loop invariants using the Houdini framework for (monomial) predicate abstraction [Flanagan and Leino 2001]. Specifically, we consider predicate templates of the form $x_i = x_j$ relating values of the same variable from different program versions, and compute the strongest conjunct that satisfies the conditions of rule (5) of Figure 7.

规则 (5)：对于循环，本文采用了谓词抽象的方法，选取一组形如 $x_i = x_j$ 的（单项）谓词模板（用来关联起多个程序版本间的变量），再用 Houdini 算法求出归纳循环不变式。

规则 (1) 与规则 (6)：出现了表示乘积构造的 $\circledast$ 符号。其中规则 (1) 仅对第一个 hole 对应的 edits 构造乘积，规则 (6) 则表示对任意的一个完整程序结构构造乘积。构造乘积程序是组合式推理不满足推理条件（如规则 (4) 分支结构前条件不蕴含程序执行相同分支）或（由规则 (2) 的上近似导致的）不完备时采取的 fallback。


![[product construction.png|697]]

图 8 具体介绍了如何构造乘积，本质上也是在设计启发式的对齐策略。

论文证明了图 7 的关系后置条件推断和图 8 的乘积构造的 soundness。

## 实现与不足

> As standard in prior verification literature [Flanagan et al. 2002], we model each field $f$ in the program as follows: We introduce a map $f$ from object identifiers to values and model reads and writes to the map using the $select$ and $update$ functions in the theory of arrays. Similarly, our implementation models collections, such as `ArrayList` and `Queue`, using arrays. Specifically, we use an array to represent the contents of the collection and use scalar variables to model the size of the collection as well as the current position of an iterator over the collection [Dillig et al. 2011].

对数据结构（映射，数组，集合）的建模：数组理论。

> our implementation checks semantic conflict freedom on the method’s return value, the final state of the receiver object as well as any field modified in the method.

在实现中，分析的对象是 Java 程序，可观测输出状态 $out$ 被设置为函数返回值，reciever 对象以及被改变的类成员对象。

不足之处：
- 做的是函数粒度的分析，且做了很多假设（函数签名，变量名等不会改变；外部调用保持 semantic conflict freedom）
- 没有讨论并发， 终止性，异常控制流