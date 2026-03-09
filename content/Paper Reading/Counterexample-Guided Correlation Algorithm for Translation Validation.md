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

本工作做的是（从 C 语言翻译得到的） LLVM IR 程序（本文用 C 表示）与（从 LLVM IR 继续翻译得到的） x86 汇编程序（本文用 A 表示）之间的翻译验证，因此上述概念是定义在不同层次程序之间的。

![[image-20260306015214766.png|540]]
> 
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

---
与 [[Semantic Program Alignment for Equivalence Checking|PLDI'19 SPA]] 的对比讨论：

> They first "guess" an alignment predicate (AP) that must hold at all nodes of the required product-CFG. Then, using concrete execution traces (on identical inputs) on C and A, they construct a candidate product-CFG (which they call the Program Alignment Automaton or PAA) -- the execution traces are employed to determine potentially correlated transitions by identifying a correspondence between PCs such that the machine states satisfy the guessed AP at those correlated PCs. By construction, the language accepted by the PAA includes the concrete execution traces (for all available inputs) on C and A. In other words, the PAA represents the product-CFG as *guessed* through the available concrete execution traces. The primary idea is to extrapolate the behavior of the two programs on a small set of concrete traces by using a "good" AP guess, to all possible executions on C and A. This approach is best-effort because: (a) it requires execution traces with adequate path coverage on both C and A; we find that adequate coverage may require traces that exhibit an exponential number of distinct behaviors. (b) It relies on a good AP guess: an AP that is too strong would ignore the desired PAA while an AP that is too weak would result in too many satisfying PAAs, of which most would be incapable of yielding a provable bisimulation.

> To understand SPA through an example: if for a given input, program C takes path $\eta_{C}$ and program A takes path $\eta_{A}$ such that the AP is satisfied by the machine states of C and A at the endpoints of $\eta_{C}$ and $\eta_{A}$ respectively, then a PAA transition (product-CFG edge) that correlates the two paths, $\eta_{C}$ and $\eta_{A}$, is added to the PAA. In other words, an edge $e = (\eta_{C}, \eta_{A})$ is added to the PAA only if the states at the two (start and stop) endpoints are related by AP and the programs C and A are known to take these paths $\eta_{C}$ and $\eta_{A}$ respectively for the same input (for all available concrete inputs). By construction, the start nodes ((C0,A0) in fig. 1) and exit nodes (EC, EA) of the two programs are always correlated in the PAA. Inductive invariants are then inferred on the PAA (which is identical to a product-CFG) and the equivalence proof is completed if the inferred invariants guarantee observable equivalence.

> Our first criticism of this approach is that **a path is correlated in the PAA only if it is seen to be taken in one of the concrete execution traces**.

只有出现在测例轨迹中的路径才会被用来构造 PAA 的转移

![[Paper Reading/Counterexample-Guided Correlation Algorithm for Translation Validation__assets/Figure 2.png|522]]

考虑图 2 的例子，(a) 中的 C 程序是一个循环，循环内有三条分支。(b) 中的汇编程序进行了向量化优化，一轮迭代可以计算 C 循环的 4 轮迭代，且消除了条件分支。要用 SPA 去证明等价性，需要将汇编中的一条路径与 $3^4 = 81$ 条对应的 C 路径对齐。(c) 中的边采用了 series-parallel digraph representation 来表示程序路径，用 + 表示 parallel composition，用 - 表示 series composition）

> Our second criticism of the SPA approach is that **it depends heavily on the availability of the required AP (alignment predicate).**

简单理解就是 AP 的选择对于对齐效果的影响非常大，很难去一次性猜到这个最合适的 AP。

![[Paper Reading/Counterexample-Guided Correlation Algorithm for Translation Validation__assets/Figure 3.png|543]]

图 3 的这个例子，(a) 中 C 程序和 (b) 中的汇编程序都有三个循环（外层，内层 1，内层 2，其中内层第一个循环在汇编中被 4:1 向量化）。一个良好的 Alignment Predicate 需要具备  `(i = r1) /\ ((j = r2) \/ (j = r5))` 的形式。
假如 AP 太强，比如 `(i = r1) /\ (j = r2)`，就对不上内层的第二个循环。假如 AP 太弱，则会产生大量虚假的 correlation，比如将内层第一个循环与外层循环 1:1 对齐，从而无法找到满足条件的不变式。

> (1) We use a **series-parallel digraph representation**(即图 2.(c) 以及图 3.(c)的转移边上的表示形式) for pathsets to correlate them efficiently across C and A in a single step. This representation enables **linear-sized SMT proof obligations** while determining correlations across pathsets, even when a single pathset may contain an exponential number of individual paths. The discharge of linear-sized SMT proof obligations, that are generated due to the control-flow transformations performed by a typical compiler, is typically fast, even though it remains worst-case exponential-time. In contrast to this approach, an algorithm that attempts to correlate each individual path separately (for an exponential number of paths) would require an exponential amount of time, even for computing equivalence across trivial transformations. 
> (2) The Counter algorithm **incrementally constructs the product-CFG through counterexample-guided pruning and ranking subprocedures, and does not require an alignment predicate**.

---

在 C 中引入特殊的 error 状态，表示 UB。

用 weak trace equivalence 来定义 C 和 A 的等价性：对于任意输入，满足以下任一条件：
1. 两个程序产生相同的 observable events (包含 return values, heap states, intermediate calls to undefined procedures)
2. C 在该输入上触发 UB

