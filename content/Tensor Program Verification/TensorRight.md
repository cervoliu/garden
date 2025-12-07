---
title: TensorRight
author: Cervol Liu
date: 2025-11-13
tags:
  - tensor
---

#tensor #verification #POPL25

Core insight:
achieve unbounded verification by proving that there exists **a bound on tensor ranks**, under which bounded verification of all instances implies the correctness of the rewrite rule in the unbound setting. 

用 bounded-verification 验证 bound 内的 rank，再 k-induction on rank 完成 unbound 的验证

we extend these observations to any arbitrary rule **by first partitioning the axes of a tensor into “groups”, where all axes in a group (aggregated-axes) share the same “role” and are treated uniformly by the operators**. We then present an algorithm to compute a sufficient rank for each group, allowing us to avoid verifying the rule for ranks beyond these sufficient ranks.

以聚合轴的概念表示任意 rank 的张量

Contributions:
- 提供了一个 DSL，支持用户表达 tensor compiler 中的 tensor graph rewrite rule
	- 支持 rank- and size-polymorphic。rank-polymorphic 是通过 aggregated-axes 以及 rank class 两个概念实现的。每条规则中 aggregated-axes 以及 rank class 由用户定义。
	- 支持 precondition
- 形式化定义了 DSL 的 denotational semantics
- 自动验证用户书写的规则
- 验证了张量编译器 XLA 代数化简模块中的重写规则 （115/175）

Cons: 
- 只支持验证仅包含 **layout-insensitive** operators 的规则
- 对 reduce operator 的化简处理是启发式的，且需要人工提供 hints，半支持

***
##  §2 background and motivation

### §2.1 preliminary

概念自查: tensor, axis(dimension), rank(dimensionality), shape, size, operator attributes

算子参数由参与运算的 tensor 和 operator attributes 组成

![[dyup-slice example.png]]

***

## §3 overview

### §3.1 TensorRight Rewrite Rules
考虑如下重写规则

$$
\text{dy-slice}(Y,B,L) \implies_{\!\!\! E-B'=L \; \land \; P=1 \; \land B'=B} \; \text{slice}(Y,B',E,P) 
$$

dy-slice 算子从输入张量 Y 的起始坐标 B 处提取长度为 L 的子张量（B，L 都是向量）
slice 算子从输入张量 Y 的起始坐标 B' 到终点坐标 E 以 P 为步长提取子张量（B', E, P 都是向量）
重写规则的 precondition 表达了该规则成立的条件
![[Fig. 4. Illustration of dy-slice and slice operators..png|| 400]]

### §3.2 Representation in TensorRight DSL

用户可以将上述重写规则以 TensorRight 的 DSL 写出：
```haskell
1 rule = do
2     rcls <- newRClass "rcls"
3     [size, start, start', length, end, stride] <-
4     newMaps ["size", "start", "start'" "length", "end", "stride"] rcls
5     Y <- newTensor @TensorInt "Y" [rcls --> size]
6     lhs <- dynamicSlice Y [rcls --> start] [rcls --> length]
7     rhs <- slice Y [rcls --> start'] [rcls --> end] [rcls --> stride]
8     precondition [end, start', length] $ \[end, start', length] -> end - start' .== length
9     precondition [stride] $ \[stride] -> stride .== 1
10    precondition [start, start'] $ \[start, start'] -> start' .== start
11    rewrite "DynamicSlice(Y) => Slice(Y)" lhs rhs
12
13 verifyDSL rule
```
这个 DSL 是 rank-polymorphic 的，意思是说它能够抽象地表示任意 rank 的张量：
- aggregated-axes 具有 abstract rank，它是针对特定算子而言的。根据算子对张量不同 axis 的不同作用，可以将张量的轴划分成若干个 aggregated-axes，每个 aggregated-axes 包含若干个 axis，它们在该算子下的 role 是相同的。
- rank class 表示 aggregated-axes 实例化的限制：某一些 aggregated-axes 必须实例化成相同的 rank（它们属于同一个 rank class）。
![[Definition. Rank Class.png]]
- 这里，由于 dy-slice 和 slice 算子对于参数张量 Y 的每个 axis 的作用是相同的，因此只有一个 aggregated-axes，也就只对应一个 rank class (这里都用 “rcls” 表示)。
- size, start, length, end, stride 可以视为 domain 为 rcls 的 map，将 rcls 表示的 aggregated-axes 内的每个 axis 映射到整数

```haskell
[rclass0, rclass1] <- newRClasses ["rclass0", "rclass1"]
size0 <- newMap "rclass0Size0" rclass0
size0' <- newMap "rclass0Size1" rclass0
size1 <- newMap "rclass1Size" rclass1
let tensorShape = 
[
	rclass0 '-->' size0 '@@' "label0", 
	rclass0 '-->' size0' '@@' "label1",
	rclass1 '-->' size1
]
x <- newTensor @TensorInt "x" tensorShape
```
这段代码定义了一个有三个 aggregated-axes 的张量 x，其中两个 aggregated-axes 属于 rclass0，一个属于 rclass1。两个属于 rclass0 的 aggregated-axes 虽然 rank 相同，但允许有不同的 size。

### §3.3 Verification
![[Fig. 5. TensorRight Overview and Workflow.png]]
- 用户将待验证的重写规则用 TensorRight DSL 表示出来
- Bound inference: 为每个 rank class 推断出 sufficient bound
- 实例化每个 rank class 到具体的 rank，符号执行 DSL 得到语义，SMT 求解
- 如果 bound 内的所有实例验证通过，则可以通过 k-induction 归纳证明更高 bound 的情况

***
##  §4 rewrite rule representation

### §4.1 具名轴（named-axes）

概念：named-axes: give explicit names to the axes of a tensor

shape 可以视为从 name 到 sizes 的映射，access 可以视为从 name 到 indices 的映射

例子：dot 算子（类似于 `numpy.tensordot`）。 $\mathrm{dot}(t_1, t_2, \{a_2, a_3\}, \{a_4\})$，$t_1,t_2$ 为参与 dot 运算的两个 tensor，$\{a_2,a_3\}$ 称为 contraction axes，$\{a_4\}$ 称为 batch axes。假设 $t_1,t_2$ 的 shape 分别为 $\{a_1 \mapsto 2, a_2 \mapsto 3, a_3 \mapsto 4, a_4 \mapsto 5\}$ 以及 $\{a_5 \mapsto 6, a_2 \mapsto 3, a_3 \mapsto 4, a_4 \mapsto 5 \}$。dot 算子会对沿着 contraction axes 求和，最后这些轴被“收缩”（类比矩阵乘法）。batch axes 为共享但不收缩的轴，可以被并行地处理。运算结果的张量的 shape 是 $\{a_1 \mapsto 2, a_4 \mapsto 5, a_5 \mapsto 6\}$。
有时，需要对 axes 重命名。例如 dot 算子的例子中，对于既不在 contraction axes, 也不在 batch axes 中的轴（称为 spatial axes），就需要重命名来保证运算结果中的 named-axes 没有重名。

对于 layout-insensitive operators，可以将张量的 named-axes 视为无序集合。
对于 layout-sensitive operators (e.g. `reshape`, `bitcast`)，这些算子的语义和张量的物理布局是相关的（见 [[Layout sensitive operator in tensor algebra]]），不能将张量轴视为无序集合。后续定义的 aggregated-axes 事实上假设了“轴的无序性”（不考虑对轴的顺序的建模），否则会使得验证 unbounded rank 过于复杂。
### §4.2 聚合轴（aggregated-axes）

aggregated-axes: 一个特定算子对参与运算的张量的不同 axis 的作用效果不同，按照不同的作用效果将 named-axes 划归到若干 aggregated-axes 中。 例如 dot 算子的例子中，所有的 axes 都可以划分到 contraction axes, batch axes, spatial axes 这三种 aggregated-axes 中的一种。这些 aggregated-axes 可以被**实例化**成不同 rank 的 concrete named-axes 的集合，方便用有限个数的 axes 表示任意 rank 的 tensor。

在这个视角下，tensor 的 shape 和 access 都可以定义成一个嵌套映射（称为 aggregated-map），外层先将 aggregated-axes 映射到对应的 named-axes 集合，内层再将每一个 axis 映射到一个整数：
![[aggregated-map.png]]
可以在 shape 和 access 之间定义 element-wise 意义下的运算。

![[Core rewrite rule representation with selected operators.png|| 600]]

***
## §5 denotational semantics

对 `HLA-HLO` 中的算子给出指称语义，domain 是张量（将 access 映射到 element）。

算子的指称语义要求定义算子作用后的结果张量，即一个从 Access 到 Element 的映射：

![[Denotational Semantics of some core operators.png]]

一些其他的算子也可以表示成既有算子的组合，例如：
![[dot operator.png]]
在上面的 dot 算子的那个例子中，$S_1 = \{a_1 \mapsto 2, a_2 \mapsto 3, a_3 \mapsto 4, a_4 \mapsto 5\}$，$S_2 = \{a_5 \mapsto 6, a_2 \mapsto 3, a_3 \mapsto 4, a_4 \mapsto 5 \}$，$X_c = \{a_2, a_3\}, X_b = \{a_4\}$。
可以将 dot 算子的语义用 reduce, expand, binary 的组合来表达（先将两个输入张量 expand 到相同的 shape，用乘法二元运算作用，再 reduce 掉 $X_c$ ）。

这里面比较特殊的两个算子是 **Reduce** 和 **Relabel**

***
### $\mathrm{reduce}(\oplus, e, X)$ 

![[reduce operator.png]]

其含义是：
1.  输入: reduce 算子接收一个二元运算符 $\oplus$，一个张量 $e$ ，以及 $e$ 的一个聚合轴的集合 $X$ 作为输入。集合 $X$ 中的聚合轴要使用 $\oplus$ 归约掉。
2.  输出: reduce 算子返回一个新的张量。
    *   这个新张量的形状是 $S \setminus S|_X$，这意味着所有在 $X$ 中的聚合轴都被移除了。
    *   由于 $X$ 是聚合轴，其 rank 是不确定且无界的，因此不能用一般的求和式来表示运算结果，故引入 Reduction Element 作为未解释元素（每一种可能的实例化对应一种解释）来抽象地表示无界规约的结果。结果张量中的每个元素都是一个 Reduction Element。
***
一个 Reduction Element 的表示形式是：
$$
\mathrm{Red}^{\oplus}_{I_0, \dots, I_k} f(\{x_0 \mapsto I_0, \dots, x_k \mapsto I_k \})
$$
其中：
*   $\oplus$ 是一个二元运算符，代表归约时使用的操作（例如，求和、求最大值等）。
*   $I_0, \dots, I_k$ 是符号化规约索引（reduction indices），每一个 $I_j$ 都是一个符号化的映射，从聚合轴 $x_j$ 内的具名轴到符号化的整数索引。
*   $X$ 是被归约的聚合轴（aggregated-axes）的集合，$X=\{x_0, \dots, x_k\}$。
*   $f$ 将（具体的）规约索引映射到张量对应位置的值，形式化地，$f=\lambda i_0, \dots, i_k. [\![e]\!] [\{x_0 \mapsto i_0, \dots, x_k \mapsto i_k\} \cup A]$。这里的 $A$ 可以认为是一个常量，表示不在 $X$ 中的其他轴的具体索引。
* $[\![e]\!] [\{x_0 \mapsto I_0, \dots, x_k \mapsto I_k\} \cup A]$ 表示在给定实例化下 $f$ 值的集合，形式化地，$[\![e]\!] [\{x_0 \mapsto I_0, \dots, x_k \mapsto I_k\} \cup A] = \left\{ f(i_0, \dots, i_k) \mid \{x_0 \mapsto i_0, \dots, x_k \mapsto i_k\} \cup A \in \mathrm{Access}(e) \right\}$，简记为 $f(\{x_0 \mapsto I_0, \dots, x_k \mapsto I_k \})$。
* $\mathrm{Red}=\lambda I_0,\dots, I_k. \bigoplus f(\{x_0 \mapsto I_0, \dots, x_k \mapsto I_k \})$。
***
回到 reduce 算子。

在 XLA 的重写规则中，reduce 算子的出现位置不一定总是在最外层，例如以下的重写规则：
$$
\mathrm{reduce}(\mathrm{concat}(\mathrm{reduce(A), \mathrm{reduce}(B)})) = \mathrm{reduce}(\mathrm{concat}(A, B))
$$
这样可能会出现 reduction element 之间运算甚至嵌套的情况，为了避免这种复杂的语义，TensorRight 设计了三条针对 reduction element 的化简规则：
![[rules on reduction elements.png]]
如果这三条化简规则无法化简，TensorRight 会报告该重写规则无法验证。 

如此，TensorRight 将所有涉及 reduce 的重写规则的验证全部规约成判定两个 reduction element 是否相等，即判定 $\mathrm{Red}_{X}f(X)=\mathrm{Red}_{Y}g(Y)$ 。这里的 $f(X)$ 和 $g(Y)$ 都是无界的集合。

本文的做法是验证一个更强的条件，即 $f(X)$ 与 $g(Y)$ 表示相同的 multiset。要验证两个无界的集合是相同的，只需在集合之间找到一个双射，使得双射两边的元素在任意的张量实例化以及算子属性下都对应相等。

这部分的验证需要由用户提供一组 X 与 Y 之间的 relation 作为 hint，TensorRight 将其翻译成 SMT 约束（1. relation 确实构成双射 2. 双射两边的元素对应相等）。

p.s. 这里有一个细节上的问题，X 与 Y 是两个 tensor 的聚合轴的集合，它们可能会有重名的轴。并且由于 Red 是未解释元素，我们每次写一个 Red 时，对应的 $I_0, \dots, I_k$ 都是新的符号变量。这为用户写 hint 带来困难。为此，TensorRight 将 symbolic names（$RI_0, \dots, RI_k$）的信息加入到 reduce 算子的参数中：

![[reduceSI operator.png]]

***
### $\mathrm{relabel}(e,R)$

![[relabel operator.png]]
这里的 $R$ 是 [[Core rewrite rule representation with selected operators.png]] 中定义的 Relabel Maps，定义了聚合轴集合上的置换。 Relabel 是作者为了验证的方便而引入的。

根据作者的说法，在 TensorRight 中，由于采用了 aggregated axes 的表示，轴是无序的，在这个意义下一些作用于轴顺序的算子（如转置）相当于 idendity，然后 relabel 是用来对类似于 $t+\mathrm{transpose}(t)$ 的表达式做轴的重命名的，例如：

![[transpose rewrite rule example.png]]
但我个人的理解是，这类作用于轴顺序的算子，在 TensorRight 的语义下就可以直接定义成特殊的 relabel 算子，例如转置就是一个二轮换，是一种特殊的置换。

## §6 Verification of Rewrite Rules

### §6.1 Overview of the Verification
对于一条重写规则 $\text{LHS} \Rightarrow_{C} \text{RHS}$，需要验证：

$$
\forall v\in vars, C\land \text{valid-expr(LHS)} \to [\![\text{LHS}]\!] = [\![\text{RHS}]\!]
$$

这里的难点是张量的 rank 与 size 都是无界的。
考虑固定一组张量的实例化 $I$，那么要验证在所有**合法的**实例化 $I$ 下，上式成立。这个时候 rank 是有界的，但 size 是无界的。然而 size 无界并不是问题（因为 size 本质上只是一个 SMT 变量或者说是映射，只要我们知道了 rank，就知道了 size 的类型）。

问题转化成：
1. 如何判定一组实例化是否合法
2. 如何验证无限多的实例化的情况

对于问题 1，实际上这部分是由用户定义 Rank Class + DSL 前端的 Type/RClass Check 来保证的。也就是说，用户在用 DSL 写 rewrite rule 的时候，给每个 aggregated axes 分配合适的 rank class，保证它们在后续算子运算中不会出现 rank/size 不匹配的情况（类似于声明变量类型）。

对于问题2，简单起见先考虑只有一个 aggregated axes $x$ 及其 rank class $c$ 的情况。如果能找到一个 rank bound $k$，使得 
$$
\forall i\geq k,\; \text{valid}(i) \to \text{valid}(i+1)
$$
那么 $k$-induction on rank，就有

- Basis: 
$$
\bigwedge_{i=1..k} \text{valid}(i)
$$
- Induction: 
$$
\text{valid}(k) \; \land \; \left[ \forall i\geq k, \; \text{valid}(i) \to \text{valid}(i+1) \right] \implies \bigwedge_{i \geq k} \text{valid}(i) 
$$
就能完成无界情况的证明（将无界的验证转化为 bound 内的验证）。所以证明的关键就在于找到一个充分的 bound。

### §6.2 Bound Computation Example

以 `PadLowCombine` 规则为例：
$$
\text{pad-low}(\text{pad-low}(Y,0,L_1),0,L_2) \Rightarrow_{L_1\geq 0 \; \land \; L_2 \geq 0} \; \text{pad-low}(Y,0,L_1+L_2)
$$
where $L_1=\{x \mapsto l_1\}$ and $L_2 = \{x \mapsto l_2\}$, for some maps $l_1$ and $l_2$

pad-low 算子语义如下：

![[pad-low operator.png]]

由于 LHS 和 RHS 具有相同的 shape，因此 $[\![\text{LHS}]\!] = [\![\text{RHS}]\!]$ 可以对所有合法的 $A \in \text{Access}(\text{LHS})$ 展开，并按照语义符号执行，得到最终的语义：

![[symbolic evaluation example.png]]

