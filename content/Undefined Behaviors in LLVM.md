---
tags:
  - LLVM
  - undefined-behavior
---
可以分类成两种 - immediate UB 和 deferred UB:
- 通常简称的 "UB" 一般指的是 immediate UB，是最强形式的未定义行为。immediate UB 用来表示在 CPU 上 trap 的操作（例如除以0，空指针解引用）。在编译过程中如果遇到 UB 一般会直接终止。
- Deferred UB:
> 	Deferred UB is a lighter form of UB. It enables instructions to be executed speculatively while marking some corner cases as having erroneous values. Deferred UB should be used for cases where the semantics offered by common CPUs differ, but the CPU does not trap.
	- 体现在 IR 中的形式为特殊的值（**poison** 与 **undef**）
	- **undef**: 表示某种特定类型的任意值，仅用于表示 loading non-initialized memory 的语义
	- **poison**: 被污染的值，可被算数运算传播，类似于 NaN
> 		Poison values are a stronger form of deferred UB than undef. They still allow instructions to be executed speculatively, but they taint the whole expression DAG (with some exceptions), akin to floating point NaN values.
- deferred UB 可以转化为 UB: 
	- branching on **undef** or **poison**
