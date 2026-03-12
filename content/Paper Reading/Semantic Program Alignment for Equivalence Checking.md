---
tags:
  - equivalence-checking
  - PLDI19
  - paper-reading
zhihu-title: "[PLDI'19] Semantic Program Alignment for Equivalence Checking"
feishushare: true
feishu_url: https://feishu.cn/docx/BjQcd4HwaolIAGxn9wecSqjNntC
feishu_shared_at: 2026-03-09 12:46
zhihu-topics:
  - PLDI
  - 等价性验证
---
## 概要

> Program equivalence checking is commonly performed in two stages: the first stage is to construct a *product program* for the two programs by *aligning* them, and the second is proving a safety property, or *invariant*, of the resulting program [4, 40].

程序等价性的证明通常通过两步完成：第一步是通过**对齐**的方式构造关联起两个程序的**乘积程序**；第二步是在乘积程序上，用经典的归纳不变式的方法完成等价性的证明。
第二步沿用了关系程序验证中经典的构造关系不变式以验证关系属性的技术路线，在等价性验证的任务中本质上并无特殊之处，只是将程序等价性看成一种特殊的关系属性。因此，对于等价性验证，解决问题的关键在于第一步，即找出更好的程序对齐点（对齐谓词），从而构造出验证高效的乘积程序。

![[product-programs.png|403]]

图 1 展示了构造乘积程序的一个 motivating example。对于子图 (a)(b) 中的函数 $f, g$，(c) 给出了平凡的乘积构造，即简单的首尾拼接，然而 (c) 中的乘积程序 X 对于证明等价性没有提供任何帮助，因为分析 X 在本质上与分别对 $f, g$ 分析并无二致，其依然需要对 $f, g$ 中函数的完整概括（inductive loop invariant）。而 (d) 中的乘积程序 Y 则给出了更为精细的乘积构造，这种构造利用了 $f, g$ 中的 while 循环迭代次数相同的语义性质（其实图中的循环条件 `*` 与循环体 `A`, `B` 并不能保证这一点，但论文就是这么写的，我们假设该性质成立），从而将问题转化为证明一个相对更容易的关系不变式 $Inv$。

> Given two functions $f$ and $g$ along with test cases provided by the user, we build a *trace alignment*, which is a pairing of states in execution traces of $f$ and $g$ for each user-provided test case. Constructing the trace alignment is guided by the selection of a weak invariant, called an *alignment predicate*, that identifies pairs of machine states that should be aligned. Only once we have identified a trace alignment based on semantic properties of the programs do we lift the alignment back to the program syntax, construct a product program, and learn invariants that we attempt to prove.

有别于以往基于语法的对齐技术，本文提出了一种语义驱动的程序对齐技术。该技术通过用户提供的一组测例，配合启发式的对齐谓词，先构建起（定义在程序状态上的，语义级的）轨迹对齐（trace alignment），并借此进一步构建（定义在程序位置上的，语法级的）程序对齐自动机（PAA，其唯一对应一个乘积程序）。

后续的证明步骤则与以往工作类似，在 PAA 上（用数据驱动的方法，从用户给定的测例中）学习不变式，并检查不变式的归纳性。

---
## 详解

### 基本定义

- 给定待验证的两个函数 $f$, $g$，以及一组测例 $\tau_1, \dots, \tau_n$。
- 将函数 $f$, $g$ 表示为 CFG 的形式。CFG 的结点为程序位置（简称为程序点，用 $q$ 表示 $f$ 的程序点，$q'$ 表示 $g$ 的程序点，后同），CFG 的边为带有控制流守卫条件（guard predicate）的基本块。不失一般性地，认为函数 $f$, $g$ 有唯一的入口点 $q_0, q'_0$  和出口点 $q_{exit}, q'_{exit}$。
- 定义程序**路径**（path，用 $P, Q$ 分别表示 $f, g$ 的路径）为基本块的序列。
	- 定义路径条件为路径所包含基本块的守卫条件的合取。
- 定义程序**状态**（state，用 $\sigma$ 表示）为 {程序位置，全体寄存器，栈内存，堆内存} 的一组取值。
	- 称初始状态为程序位置为入口点的状态，末尾状态为程序位置为出口点的状态。
	- %%称输入状态为初始状态的取值在 {输入寄存器，堆内存} 下的投影。%%
	- 称输出状态为末尾状态的取值在 {返回值寄存器，堆内存} 下的投影。
	- 测例对应于初始状态的具体值实例化。
- 将测例 $\tau$ 作为函数的初始状态，可执行出一条程序状态的序列，称为执行**轨迹**（trace，简称轨迹或迹，用 $\sigma$ 表示）。
- 等价性的定义：称两个 `x86-64` 函数 $f$, $g$ 等价，当且仅当 $f, g$ 以任意相同的初始状态（除程序位置外）运行，满足以下任一条件：
	1. 两个函数均正常终止（从出口点返回），且具有相同的输出状态。
	2. 两个程序均非终止，或遇到相同的运行时错误。

### 技术方案

![[Paper Reading/Semantic Program Alignment for Equivalence Checking__assets/Figure 4.png|543]]

对于**对齐谓词**（alignment predicate, AP） $\xi$ 以及 $f,g$ 在相同测例下得到的迹 $\rho,\rho'$，令 $\sigma \in \rho, \sigma'\in \rho'$ 为迹的某个程序状态。若 $\xi(\sigma,\sigma')$ 成立，则称迹 $\rho,\rho'$ 在状态 $\sigma,\sigma'$ 处由 $\xi$ 对齐。特别地，认为 $\rho, \rho'$ 在程序初始状态和末尾状态处对齐（即使 $\xi$ 不成立）。所有由 $\xi$ 对齐的状态对组成**轨迹对齐**（trace alignment），它是状态的多对多映射。

如图 4，选择对齐谓词为 $\xi \triangleq array + 4i = array'$，左右两张表格展示了 $f, g$ 在相同测例下的轨迹。两张表格之间相连的边 $e$ 表示由 $\xi$ 对齐的状态对，它们组成 $\xi$ 的轨迹对齐。

![[Figure 5.png|314]]

轨迹对齐可以诱导出对应路径（corresponding paths），如图 5。考虑轨迹对齐中不相交（do not cross each other）且相邻（have no edges in between them） 的两条边。例如，考虑 $e_{21} \to e_{42}$，$f$ 从 $e_{21}$ 到 $e_{42}$ 的路径为 $bb$，$g$ 从 $e_{21}$ 到 $e_{42}$ 的路径为 $c'$，则称 $bb$ 和 $c'$ 构成一组对应路径。

> We initialize the PAA with a node for every pair of program points in the two programs. We consider pairs $(\rho, \rho') \in \text{TA}$ along with minimal $\nu,\nu'$ such that $(\rho\nu,\rho'\nu')\in \text{TA}$ (e.g. for the trace alignment in Figure 4, we consider the pairs shown in Figure 5). For each such pair we add a transition $(p,p') \to (q,q')$ labeled by the paths of basic blocks taken by $\nu$ and $\nu'$, where $(p,p')$ is the last pair of program points in $(\rho,\rho')$ and $(q,q')$ is the last pair of program points in $(\nu,\nu)$. As an optimization, we consider only $(\nu,\nu)$ that are small, for example, fewer than 10 machine states in length.

用**程序对齐自动机**（Program Alignment Automaton，PAA）表示附有对齐信息的乘积程序：
- PAA 的结点为程序位置的二元组 $(q,q')$。
- PAA 的转移边形如 $(u,u') \xrightarrow{P,Q} (v,v')$，其中 $P$ 为 $f$ 的一条从 $u$ 到 $v$ 的路径，$Q$ 为 $g$ 的一条从 $u'$ 到 $v'$ 的路径，且 $P,Q$ 不同时为空。
- 对于每组对应路径 $P,Q$，在 PAA 中加入标注 $P,Q$ 的转移。例如，对于上述分析的 $e_{21} \to e_{42}$，其对应的 PAA 转移为 $(q_2,q'_2) \xrightarrow{bb,c'} (q_2,q'_2)$。

![[Paper Reading/Semantic Program Alignment for Equivalence Checking__assets/Figure 3.png|412]]
将图 5 构造的 PAA 化简（见[[#PAA 的化简]]），可以得到如图 3 所示的 PAA。

在 PAA 的每个结点 $s$ 处学习一个不变式 $\phi_s$。特别地，对于入口结点，令不变式为「断言输入状态相等」；对于出口结点，令不变式为「断言输出状态相等」。对于其他结点，不变式由运行测例得到的轨迹学习得到。

![[Figure 7.png|420]]

用关系 Hoare 三元组表示证明义务：令 $\phi_1, \phi_2$ 表示 $f$ 和 $g$ 的状态上的谓词（不变式），$P$, $Q$ 分别表示 $f$, $g$ 的一条执行路径，则 $\{\phi_1\} \; P; Q \; \{\phi_2\}$ 表示如下陈述：若在状态 $\sigma,\sigma'$ 处 $\phi_1(\sigma,\sigma')$ 成立，且执行路径 $P,Q$，则执行终止于状态 $\sigma'',\sigma'''$，且 $\phi_2(\sigma'',\sigma''')$ 成立。

用 PAA 证明程序等价性：
1. 对于每条转移边 $s \xrightarrow{P, Q} t$，证明 $\{\phi_s\}\; P;Q \; \{\phi_t\}$（i.e. 转移的归纳性）。
2. 对于每个结点 $s = (u,u')$ ，所有不在 PAA 中的从 $(u,u')$ 出发的程序路径对均不可达（即，PAA 没有遗漏乘积程序语义，是一个可靠的上近似）。形式化地，若 $P, Q$ 分别为 $f, g$ 中从 $u, u'$ 出发的路径，且 PAA 不存在转移 $s \xrightarrow{P^{*},Q^{*}} s'$ 满足 $P^{*},Q^{*}$ 分别为 $P,Q$ 的前缀，则有 $\{\phi_{s}\}\; P;Q \; \{\bot\}$ 。
3. PAA 中不存在由转移边构成的回路，使得回路的每条转移边上 $f$ 侧（或 $g$ 侧）的程序路径均为空（否则，$g$ （或 $f$）非终止）。
4. 出口结点处的不变式蕴含输出状态相等。

### 技术细节

#### PAA 的化简

![[Figure 8.png|484]]

> Removing nodes makes finding provably correct invariants easier, and removing transitions decreases the number of proof obligations.
> ...
> We perform the following two operations until we reach a fixed point.

> First, we remove every node $s$ that does not have a self-loop other than the entry and exit. ...

第一步本质上就是只保留乘积程序中的循环，将循环之间的线性控制流压缩成单条转移边。

> Second, we remove extra transitions. ...

第二步，删除冗余转移。看一下图 8 的例子就理解了。

#### 测试 PAA

假如对齐谓词选得不好，很可能得到的 PAA 会过拟合到训练数据（将测例划分为训练集和测试集，仅用训练集构建轨迹对齐）上，这时 PAA 有很大概率并不是一个合法的上近似。可以用测试集对 PAA 进行快速检查，若 PAA 不接受测试集，则立刻拒绝之，并尝试新的对齐谓词。

#### 对齐谓词

> In practice we find there is a small space of predicates that almost always contains a useful alignment predicate for pairs of equivalent x86-64 functions. Namely, choosing a predicate of the form $(c_1v_1 − c_2v_2 = k) \land \omega = \omega'$ is typically sufficient. Here $v_1$ and $v_2$ are registers or stack-allocated locations in $f$ and $g$.

对齐谓词的形式相对固定，且通过限制取值的方式进一步限制了该形式下的搜索空间。
#### 不变式

![[Figure 9.png|415]]

不变式的语言包括线性等式，不等式以及模运算意义下的等式。
对于这三种类型的不变式，文章对应介绍了三种数据驱动的学习方法。