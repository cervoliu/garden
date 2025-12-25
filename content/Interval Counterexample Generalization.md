---
aliases:
  - 区间反例泛化
tags:
  - SMT
---
区间泛化的大致思想是，根据 teacher(solver) 提供的一个反例 model，将 model 对变量赋的具体值泛化成一个区间，从而排除更多的潜在反例。

假设验证条件为 $\psi$ ，反例 model 可以表示成 $\{ x = \mu_{x}\}$ 的形式，则一种基于 **variable elimination** 的泛化方法为求解如下公式的 unsat core：
$$ \psi \land \bigwedge_{x \in \vec{x}} (x=\mu_x)$$
对于不在 unsat core 中的变量，可以直接将它们从对应的 model 中去除。可以认为 variable elimination 将这些变量的取值从 $\mu$ 泛化到了区间 $(-\infty, +\infty)$，因此也可以认为是区间泛化的一种特殊情况

扩展上述思路，引入 **boundary constraints**:
$$ \psi \land \bigwedge_{x\in \vec{x}} \bigwedge \left\{  x \geq \mu_x - d_i, x \leq \mu_x + d_j \mid 0 \leq i,j \leq m \right\} $$
其中 $\mathcal{D} = \{d_0, \dots, d_m\}$ 是刻画距离的一组递增序列，且 $d_0 = 0$。求解上述公式的 [[unsat core]]。
值得注意的是，$\psi$ 在求解时为 hard constraints，而后面的一系列 boundary constraints 为 soft constraint。

上述求解得到的 unsat core 即使在 size 上是 minimal 的，其对应的区间也不一定是最 general 的。
为此，考虑如下的 **digging generalization**:
定义 $\mathcal{I}_{x,\mu_{x},k}$ 是 $2m+1$ 个相邻区间，表示了 $x$ 的 domain，如图所示: 

![[interval digging generalization.png|500]]

并求解如下约束的 unsat core：
$$ \psi \land \bigwedge_{x \in \vec{x}} \bigwedge \{ \neg \mathcal{I}_{x,\mu_{x},k} \mid k=-\infty,-m,\dots,-1,1,\dots,m,\infty \} $$
依旧 $\psi$ 是 hard constraint，其他项是 soft constraints。对于这个查询求出的 unsat core $\{\mathcal{I}_1, \dots, \mathcal{I}_{m}\}$，$\wedge_{i=1}^{m} \neg \mathcal{I}_{i}$ 表示了若干个不相交的可以作为 counterexample 的 $x$ 的取值区间。取包含 $\mu$ 的那个区间作为 generalized counterexample。

---
参考文献：
[1] Xu Rongchen, Fei He, and Bow-Yaw Wang. “Interval Counterexamples for Loop Invariant Learning.” _Proceedings of the 28th ACM Joint Meeting on European Software Engineering Conference and Symposium on the Foundations of Software Engineering_, ACM, November 8, 2020, 111–22. [https://doi.org/10.1145/3368089.3409752](https://doi.org/10.1145/3368089.3409752).