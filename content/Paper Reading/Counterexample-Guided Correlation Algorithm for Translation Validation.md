---
tags:
  - paper-reading
  - translation-validation
  - OOSPLA20
feishushare: true
feishu_url: "https://feishu.cn/docx/EhgTdabdvoswHfxEi8ZcK4WGnBc"
feishu_shared_at: "2026-03-06 13:13"
---
%% 文章的最开头首先科普了 translation validation (代表工作 Alive2) 与 certified compilation (代表工作 CompCert) 的区别，详见 [[Translation Validation vs Certified Compilation]]。%%

> For most programs/compiler-transformations, it usually suffices to restrict oneself to **bisimilarity checking**, where the algorithm proceeds by **1.correlating the transitions (or paths) in the two programs** and **2.identifying inductive relational predicates (or invariants) between variables (state-elements) of the two programs at the endpoints of the correlated transitions** [Pnueli et al. 1998]. We call the endpoints of the correlated transitions, correlated *PCpairs*, given that they are formed by pairing two program locations or PCs of the respective programs. If these correlations and relational invariants ensure equivalent observable behavior (e.g., identical sequence of I/O events, identical return value and returned heap state), then we have obtained a proof (or witness) of equivalence (and bisimilarity). **This proof, involving correlations and invariants, can be represented either as a (bi)simulation relation [Milner 1971; Necula 2000; Pnueli et al. 1998] or as a product program [Zaks and Pnueli 2008], both of which are equivalent representations.**

本文的技术路线属于经典的构造程序对齐+在对齐结构上寻找关系不变式的两步方案，只不过本文把“构造程序对齐“这件事称为关联（correlation），并把被验证的属性称作双模拟（bisimulation）。

> 维基百科 bisimulation 词条对 bisimulation 的形式化定义：
> Given a labeled transition system $(S, \Lambda, \to)$, where $S$ is a set of states, $\Lambda$ is a set of labels and $\to$ is a set of labelled transitions (i.e., a subset of $S \times \Lambda \times S$), a **bisimulation** is a binary relation $R \subseteq S \times S$, such that both $R$ and its converse $R^{T}$ are simulations. 
> Equivalently, $R$ is a **bisimulation** iff for every pair of states $(p, q)\in R$ and all labels $\lambda \in \Lambda$:
> - if $p \xrightarrow{\lambda} p'$, then there is $q \xrightarrow{\lambda} q'$ such that $(p',q')\in R$;
> - if $q \xrightarrow{\lambda} q'$, then there is $p \xrightarrow{\lambda} p'$ such that $(p',q') \in R$.

本工作做的是 LLVM IR 程序与 x86 汇编程序之间的翻译验证，因此上述概念是定义在不同层次程序之间的。

![[image-20260306015214766.png]]

> **Observation-A**: There exists a trade-off between the amount of computational effort spent in identifying the "right" product-CFG and the effort spent in identifying the required inductive invariants. *For most programs/compiler-transformations, there **exists** a product-CFG where the required invariants (to prove equivalence) are formed by simply relating the bitvector and array values through equality, inequality, and affine relations.* This claim has been observed and assumed by multiple independent prior research efforts [Churchill et al. 2019; Dahiya and Bansal 2017a; Gupta et al. 2018], and we refer to this as Observation-A in the rest of the paper.

Observation-A: 对于绝大多数等价的编译变换，存在一种乘积构造，使得在该乘积程序上，所需完成等价证明的不变式仅由等式，不等式，仿射关系构成。

> Each edge in a product-CFG encodes correlated transition *paths* in the two respective programs. 
> **Observation-B**: *for most programs/compiler-transformations, we can bound the maximum length of a path that needs to be correlated within a single CFG edge.*

Observation-B: 对于绝大多数等价的编译变换，我们可以限制在单个 CFG 边中需要关联（correlated）的路径的最大长度。
在翻译验证中，虽然理论上两个程序之间的路径关联可能是无限复杂的，但在实际的编译器优化变换中（如循环展开、向量化等）需要关联的路径长度是有界的。这是因为编译器在进行优化时通常会施加一些限制，比如循环展开因子（unroll factor）的上限。

> **Observation-C**: It usually suffices to restrict the correlated PCs (that constitute the PCpairs) to the heads (first instruction) and tails (last instruction) of the basic blocks of one program’s CFG to the loop heads of the other program’s CFG.

Observation-C: 通常，将关联的 PC 限制在一个程序 CFG 的基本块头部(第一条指令)和尾部(最后一条指令)与另一个程序 CFG 的循环头之间就足够了。
换句话说，认为不变式推理足以弥合理想关联（捕捉了程序最细致的语义对齐）与 Observation-C 所约束的关联之间的差距。

> **Observation-D**: For a given PCpair, it is rare for an outgoing path in A to be correlated with more than one paths in C such that these (two or more) correlated paths in C have different endpoints, but not vice-versa. （A 表示编译后的汇编程序，C 表示源程序）

这 4 条观察中，Observation-A 约束了不变式的搜索空间，而 Observation-{B,C,D} 约束了 correlation 的搜索空间。


