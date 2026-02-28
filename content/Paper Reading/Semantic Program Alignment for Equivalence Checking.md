---
tags:
  - equivalence-checking
  - PLDI19
---
> Program equivalence checking is commonly performed in two stages: the first stage is to construct a *product program* for the two programs by *aligning* them, and the second is proving a safety property, or *invariant*, of the resulting program [4, 40].

程序等价性的证明通常通过两步完成：第一步是通过**对齐**的方式构造关联起两个程序的**乘积程序**；第二步是在乘积程序上，用经典的归纳不变式的方法完成等价性的证明。
第二步沿用了关系程序验证中经典的构造关系不变式以验证关系属性的技术路线，在等价性验证的任务中本质上并无特殊之处，只是将程序等价性看成一种特殊的关系属性。因此，对于等价性验证这个更具体的任务，解决问题的关键在于第一步，即找出更好的程序对齐点（对齐谓词），从而构造出验证高效的乘积程序。

![[product-programs.png]]
上图展示了构造乘积程序的一个 motivating example。对于子图 (a)(b) 中的函数 f, g，(c) 给出了平凡的乘积构造，即简单的首尾拼接，然而 (c) 中的乘积程序 X 对于证明等价性没有提供任何帮助，因为分析 X 在本质上与分别对 f, g 分析并无二致，其依然需要对 f, g 中函数的完整概括（inductive loop invariant）。而 (d) 中的乘积程序 Y 则给出了更为精细的乘积构造，这种构造利用了 f, g 中的 while 循环迭代次数相同的语义性质（其实图中的循环条件 `*` 与循环体 `A`, `B` 并不能保证这一点，但论文就是这么写的，我们假设该性质成立），从而将问题转化为证明一个相对更容易的关系不变式 $Inv$。

> Given two functions $f$ and $g$ along with test cases provided by the user, we build a *trace alignment*, which is a pairing of states in execution traces of f and g for each user-provided test case. Constructing the trace alignment is guided by the selection of a weak invariant, called an *alignment predicate*, that identifies pairs of machine states that should be aligned.

本文提出了一种基于语义的对齐。基于用户提供的输入测例，构造两个程序之间的 trace alignment。


---

等价性定义：称两个 `x86-64` 函数 $f$, $g$ 等价，当且仅当以任意相同的起始状态（包括寄存器，内存（栈，堆））从函数入口处运行，满足以下任一条件：
1. 两个函数均正常终止，且具有相同的返回寄存器值以及相同的堆内存状态
2. 两个程序均非终止，或遇到相同的运行时错误

用关系 Hoare 三元组表示证明义务：令 $\phi_1, \phi_2$ 表示 $f$ 和 $g$ 的机器状态上的谓词（不变式），$P$, $Q$ 分别表示 $f$, $g$ 的一条执行路径，则 $\{\phi_1\} \; P; Q \; \{\phi_2\}$ 表示如下陈述：若在初始状态 $\sigma,\sigma'$ 处 $\phi_1(\sigma,\sigma')$ 成立，且 $P$, $Q$ 被执行，则执行终止于状态 $\sigma'',\sigma'''$，且 $\phi_2(\sigma'',\sigma''')$ 成立。

用程序对齐自动机（Program Alignment Automaton，PAA）（间接地）表示附有对齐信息的乘积程序。PAA 的一个结点 $s$ 处携带一组（对齐的）程序位置（program counter / location）$(u,v)$ 以及 $s$ 处的不变式 $\phi_s$。%%假设 $f$, $g$ 均有唯一入口位置和出口位置，则 PAA 的起始结点和唯一终止结点可以分别被确定。%%
PAA 的一条转移边 $(u,u') \to (v,v')$ 携带一组有限程序路径 $(P,Q)$。

用 PAA 证明等价性的步骤：
1. 对于每条转移边 $s \xrightarrow{P, Q} t$，证明 $\hoare{\phi_s}{P;Q}{\phi_t}$
2. 对于每个结点 $s = (u,u')$ ，所有从 $123$。即，PAA 没有遗漏程序中的有效转移，是一个 sound 的上近似 
3. 出口结点处的不变式蕴含堆内存状态以及输出寄存器相等

---
- 给定待验证的两个函数 $f$, $g$，以及一组测例 $\tau_1, \dots, \tau_n$。
- 将函数 $f$, $g$ 表示为 CFG 的形式。CFG 的结点为程序位置（简称为程序点，用 $q$ 表示 $f$ 的程序点，$q'$ 表示 $g$ 的程序点，后同），CFG 的边为带有控制流守卫条件（guard predicate）的基本块。不失一般性地，认为函数 $f$, $g$ 有唯一的入口点 $q_0, q'_0$  和出口点。
- 定义程序**路径**（path，用 $P, Q$ 分别表示 $f, g$ 的路径）为基本块的序列。
	- 定义路径条件为路径所包含基本块的守卫条件的合取。
- 定义程序**状态**（state，用 $\sigma$ 表示）为 {程序位置，全体寄存器，栈内存，堆内存} 的一组取值。
	- 称初始状态为程序位置为入口点的状态，末尾状态为程序位置为出口点的状态。
	- 称输入状态为初始状态的取值在 {输入寄存器，堆内存} 下的投影。
	- 称输出状态为末尾状态的取值在 {返回值寄存器，堆内存} 下的投影。
	- 测例对应初始状态的具体值实例化。
- 将测例 $\tau$ 作为函数的初始状态，可执行出一条程序状态的序列，称为执行**轨迹**（trace，简称轨迹或迹，用 $\sigma$ 表示）。

![[Figure 4.png]]

- 对于**对齐谓词**（alignment predicate） $\xi$ 以及 $f,g$ 在相同测例下得到的迹 $\rho,\rho'$，令 $\sigma \in \rho, \sigma'\in \rho'$ 分别为迹的某个程序状态。若 $\xi(\sigma,\sigma')$ 成立，则称迹 $\rho,\rho'$ 在状态 $\sigma,\sigma'$ 处由 $\xi$ 对齐%%，$(\sigma, \sigma')$ 为对齐点%%。特别地，认为 $\rho, \rho'$ 在程序初始状态和末尾状态处对齐（即使 $\xi$ 不成立）。所有由 $\xi$ 对齐的状态二元组构成一组**轨迹对齐**（trace alignment），它是状态的多对多映射。

![[Figure 5.png|400]]

- 轨迹对齐可以诱导出**对应路径**（corresponding paths）。对于轨迹对齐中「相邻」的两条不相交的边
- 在 PAA 的每个结点 $s$ 处学习一个不变式 $\phi_s$（用来证明 PAA 为两个函数行为的有效上近似）。特别地，对于入口结点，令不变式为「断言输入状态相等」；对于出口结点，令不变式为「断言输出状态相等」。对于其他结点，不变式通过测例诱导的执行轨迹习得。