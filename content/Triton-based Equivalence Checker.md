Triton 是一个二进制动态分析库，主要的特性是提供了一组对各个架构（x86-64, x86, arm32, AArch64, RISC-V 32/64）的指令语义的编码（用 Triton 内部的 AST 来表示），这在二进制验证相关的一些工作[1]中被称为 machine model，这一验证手段被称为 decompilation-into-logic (DiL)。
Triton 对汇编指令语义的编码是手动实现的，事实上并没有严格的验证保障。而[2]使用了机器学习来 infer 指令的语义。

使用 Triton 的动态符号执行(也叫做混合执行，concolic execution)可以得到若干条执行路径，以及路径对应的路径条件与符号状态。

动态符号执行的原理：
	首先生成一个初始 seed（使用默认值或随机），一个 seed 描述了具体的执行前初始机器状态（CPU状态以及内存区域的具体赋值）。在该 seed 下，可以唯一确定一条汇编程序的执行路径。但与 simulation 不同之处在于，动态符号执行同时维护路径下的符号状态，同时记录所有的控制流分支。探索完一条路径后，回溯不同的分支条件，使用 SMT solver 求解产生新的 seed，从而获得更多的执行路径。

基于 Triton 的二进制程序 bounded equivalence checking 框架：
1. 首先对两个程序分别动态符号执行，得到路径集合
2. 在路径集合中对路径两两枚举，检查等价性

与[[Alive2]] 的一个不同点在于，Alive2 会先根据 LLVM bitcode 生成 CFG （Triton 是不会生成 CFG 的），然后在 CFG 上合并执行路径，将语义编码为 ite 语句嵌套表示的公式。因此，Alive2 的一个函数分析结束后对应一个大公式，其综合了 bound 之内所有执行路径的语义。
理论上，将 ite 嵌套的公式展开后，可以与各条路径的语义一一匹配。虽然两种表示本质上是等价的，但一般认为将单一的复杂的 SMT 查询分解为多次简单的 SMT 查询能够降低 solver 的负担，提升求解性能。

---

- [ ] unsat core 泛化本质是为了避免对不相容的两条 path 的冗余检查，这可不可以从根源上解决？也就是同时对两个程序做 concolic execution，即使用相同的 seed 在两个程序上执行
	- 如果程序等价，那么在 seed 这个具体初始状态下两个程序的最终状态也应该要相同。这里的检查只需要对比具体值。只有当具体值相等了，才需要进一步用 solver 检查符号状态是否等价。
	- 对于一个相同的 seed，两个程序执行出来的路径必然是相容的。假设路径条件分别为 p1, p2，将 $\neg (p_1 \land p_2)$  添加到搜索 seed 的约束，保证 solver 生成下一个 seed 的时候不会探索重复的路径对 。
	- 似乎完全避免了相容性检查这一个步骤？
- [ ] 对于论文中提出的 path relation $\preceq$ 以及其语法近似 $\precsim$，可以采用一种 Hybrid 检查策略（设置一个较小的 SMT timeout，先调用 solver 检查 $\preceq$，当发生 timeout 时退化为检查 $\precsim$ ）。
- [ ] 对于等价反例的 model，尝试使用求解更小的 model，区间泛化等手段。
- [ ] 关于 simplify: 这个与具体的编码方式强相关，目前观察的 SMT 查询公式的特征：
	- triton 使用 reference node 来表示 SSA 变量，对应到 SMT 中的建模就是嵌套的 let 节点。（这个应该是符号执行公式的共性特点）
	- 使用 ABV theory 建模内存的情况下，存在大量嵌套的 concat, store, select 节点
	- 尝试根据公式特征设计化简策略

---

[1] Roessle et al., “Formally Verified Big Step Semantics out of X86-64 Binaries.”
[2] Heule et al., “Stratified Synthesis: Automatically  Learning the x86-64 Instruction Set.“