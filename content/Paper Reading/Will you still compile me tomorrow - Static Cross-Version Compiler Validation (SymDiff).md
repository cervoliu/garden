---
tags:
  - paper-reading
  - compilation
  - translation-validation
---
本文提出一种 Cross-version compiler validator，围绕 CLR(Common Language Runtime) JIT(Just-In-Time) Compiler 展开一系列实验。cross-version validation 实验包括同一个编译器不同日期（间隔7个月的前后两个工具版本），不同的目标平台（x86 vs ARM），不同的编译场景（JIT vs MDIL），不同的优化级别之间的对比。

对于两个汇编程序，以函数粒度检查翻译验证意义下的等价性

先将汇编函数编码成 Boogie 函数：
- 整数：因为求解效率因素，用 unsound 的 mathematical integer 而不是 bit-vector 来建模
- 浮点与 status flag：有建模，文中略去
- 内存：we model memory as **a set of disjoint regions**, with **one region per stack frame, one region per heap object, and one region for static fields**. We assume that stores to one region will not affect loads from another region. We also assume that adding an offset to an address in one region produces an address in the same region. (假设了各个内存段不会 alias)
	- 编译器在对程序进行优化的时候本身会做一些 assumption，典型的例如 alias assumption（假设内存区域不重叠）。如果在做验证的时候不编码这些 assumption，很容易会得到不等价的结果。
- 函数调用：未解释函数

SymDiff 为了验证两个 Boogie 函数的等价性，做了一层 Harness

![[SymDiff Workflow.png|500]]

本文实现的工具因时间久远，不确定能够使用