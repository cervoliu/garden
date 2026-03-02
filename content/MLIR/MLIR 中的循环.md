---
tags:
  - MLIR
  - ml-compiler
feishushare: true
feishu_url: "https://feishu.cn/docx/FonldjxRtoqanexsaCxcBtKonXf"
feishu_shared_at: "2026-03-02 14:24"
---
## Overview

在 MLIR 中，循环的表示主要分布在 `Linalg`、`Affine` 和 `SCF` 这三个方言。它们处于编译 pipeline 的不同阶段，承载着不同的语义信息和优化目标：
- `Linalg` 为线性代数（**Lin**ear **Alg**ebra）方言，其抽象层次最高，包含了最广泛，最通用的循环语义
- `Affine` 为仿射方言，其抽象层次居中，主要包含的循环语义主要是代数空间上的多面体（polyhedral）
- `SCF` 为结构化控制流（**S**tructured **C**ontrol **F**low）方言，其抽象层次最低，提供最基本的语言控制流操作，常常作为 `Linalg` 和 `Affine` 的下沉目标方言

接下来我们逐一介绍这三个方言中的循环。
## `Linalg` Loop："By-Construction" 的结构化循环

Linalg 通过 `linalg.generic` 或命名算子（如 `linalg.matmul`）实现循环，借鉴了经典的“迭代（访问）”与“计算”分离的思想
- **优势**：由于它显式定义了 `indexing_maps`（输入输出如何映射到循环维度），编译器不需要复杂的依赖分析就能直接进行 Tiling 或 Fusion。

## `Affine` Loop：面向高效分析的多面体循环

在传统编译器中，依赖分析（Dependence Analysis）是最棘手的问题之一（例如，判断 `A[2*i+1]` 和 `A[j]` 是否在某些迭代中指向同一内存地址）。`affine` 方言通过引入严格的代数约束来解决这一问题：

- **仿射映射 (Affine Map)**：循环的边界（bounds）、步长（step）以及内存访问的下标，必须是循环归纳变量和外部未知常数的**仿射组合**（即线性组合加常数，如 `2 * d0 + s0 - 1`）。
- **精确的数学建模**：得益于严格的线性约束，编译器可以将嵌套循环的迭代空间（Iteration Space）建模为高维空间中的整数多面体。通过求解整数线性规划（ILP），编译器能绝对精确地判断任意两次迭代之间是否存在数据读写冲突。这是实现激进且安全的循环变换的理论基础。
### Dimension 与 Symbol 的严格隔离

在 `affine` 的表达式系统中，变量被严格划分为两类，以保证数学模型的线性：
- **Dimensions (维度, `d0`, `d1`...)**：通常指代循环的归纳变量（Induction Variables），它们在迭代空间中不断变化。
- **Symbols (符号, `s0`, `s1`...)**：在整个循环或特定作用域内保持不变的参数，例如动态的数组尺寸（如 `N`, `M`）。
仿射表达式允许 Dimension 乘以常数，也允许 Symbol 乘以常数，但**禁止 Dimension 和 Dimension 相乘，或 Dimension 和 Symbol 相乘**。这种非线性操作会破坏多面体模型的适用性。

```rust
// A 2d to 3d affine mapping.
// d0/d1 are dimensions, s0 is a symbol
#affine_map2to3 = affine_map<(d0, d1)[s0] -> (d0, d1 + s0, d1 - s0)>
```

## `SCF` Loop: 通用的结构化循环

### Loop-carried variables

在 MLIR 严格的 SSA（静态单赋值）规则下，变量一旦声明就无法修改。那么像 `sum = sum + i` 这种在循环中不断更新的变量该怎么表示？ SCF 给出的方案是 loop-carried variables（循环携带变量）：
- `iter_args`：在循环入口，将初始值（如 `sum = 0`）显式绑定到循环体的参数上。
- **`scf.yield`**：在每次循环的末尾，不使用传统的变量赋值，而是用 `yield` 把这一轮计算产生的新值“抛”给下一轮迭代，作为下一次的 `iter_args`，避免编译器去做内存读写依赖分析。

例子：对一个内存缓冲区 （`memref`对象）执行 sum reduce，可以表示如下

```rust
func.func @reduce(%buffer: memref<1024xf32>, %lb: index,
                  %ub: index, %step: index) -> (f32) {
  // Initial sum set to 0.
  %sum_0 = arith.constant 0.0 : f32
  // iter_args binds initial values to the loop's region arguments.
  %sum = scf.for %iv = %lb to %ub step %step
      iter_args(%sum_iter = %sum_0) -> (f32) {
    %t = load %buffer[%iv] : memref<1024xf32>
    %sum_next = arith.addf %sum_iter, %t : f32
    // Yield current iteration sum to next iteration %sum_iter or to %sum
    // if final iteration.
    scf.yield %sum_next : f32
  }
  return %sum : f32
}
```
### `SCF` Loop Operations

`scf` 中表示循环的 Operation 可以列举如下：
- `scf.for` (标准串行循环) ：最基础的 `for` 循环，由 `lower_bound`、`upper_bound` 和 `step` 控制。它严格按顺序执行，强依赖 `scf.yield` 串行传递状态。
- `scf.while` (条件控制循环) ：经典的 `while`/`do-while` 循环抽象。为了契合 MLIR 的区域 (Region) 概念，它被拆分成了两个部分："before" 区域负责计算循环条件并通过 `scf.condition` 决定是否流转；"after" 区域则是真实的循环体。
- `scf.parallel` (数据并行循环) ：表示循环的所有迭代都是相互独立的，可以并发执行。因为它不知道各个线程执行的先后顺序，所以不能用 `scf.yield`，而是用 `scf.reduce` 将多个并行分支的结果安全地归约（如求和、求最大值）成最终结果。

```rust
%res = scf.parallel (%iv) = (%lb) to (%ub) step (%step) init (%init) -> f32 {
  %val = load %buffer[%iv] : memref<100xf32>
  scf.reduce(%val : f32) {
  ^bb0(%lhs : f32, %rhs: f32):
    %sum = arith.addf %lhs, %rhs : f32
    scf.reduce.return %sum : f32
  }
}
```

-  **`scf.forall` (目标无关的张量并行)** ：比 `scf.parallel` 更贴近现代深度学习编译器。`parallel` 往往只处理标量数据的规约，而深度学习中处理的通常是巨大的张量，并且需要精确控制它在 GPU 等硬件上的执行方式。
	- **核心特性 1：`shared_outs` 与并发写入**。在张量计算中，不同的线程往往负责计算输出张量的不同分块（Tile）。`scf.forall` 引入了 `shared_outs` 操作符，并配合 `scf.forall.in_parallel` 和 `tensor.parallel_insert_slice`，允许不同线程并行地将自己计算好的小切片，安全地拼接到最终的大张量中。
	- **核心特性 2：`mapping` 硬件映射**。它可以通过属性直接指定并行维度对应什么硬件资源。

我们来剖析一段典型的张量分块计算（Tensor Tiling）的 MLIR 代码。这段代码展示了如何将一个大张量拆分给多个线程并行计算。

```rust
// %A 和 %B 是输入张量，%empty_out 是预先分配的输出张量占位符
%result = scf.forall (%i, %j) in (%num_threads_x, %num_threads_y) 
    shared_outs(%shared_out = %empty_out) -> (tensor<?x?xf32>) {
  
  // 1. 提取切片：每个线程根据自己的 ID (%i, %j) 取出负责计算的数据块
  %tile_A = tensor.extract_slice %A[%i, %j]... : tensor<?x?xf32> to tensor<16x16xf32>
  %tile_B = tensor.extract_slice %B[%i, %j]... : tensor<?x?xf32> to tensor<16x16xf32>
  %tile_out = tensor.extract_slice %shared_out[%i, %j]... : tensor<?x?xf32> to tensor<16x16xf32>

  // 2. 局部计算：在当前线程的切片上执行加法
  %add = linalg.add ins(%tile_A, %tile_B) outs(%tile_out)

  // 3. 终点同步与写回
  scf.forall.in_parallel {
    tensor.parallel_insert_slice %add into %shared_out[%i, %j]... 
      : tensor<16x16xf32> into tensor<?x?xf32>
  }
} { mapping = [#gpu.thread<y>, #gpu.thread<x>] } // 4. 硬件映射
```

`scf.forall` 在张量分块并发场景下的核心机制归纳如下：
- **硬件拓扑映射 (`mapping`)**：`(%i, %j)` 定义了虚拟的并发执行网格。`mapping` 属性显式指示编译器将逻辑上的循环维度直接绑定至具体的物理执行单元（如 GPU 的 Thread 维度），完成从软件并发到硬件并发的映射。
- **维持 SSA 语义的输出抽象 (`shared_outs`)**：规避了多线程环境下的裸指针共享。通过将外部预分配的 `%empty_out` 绑定至内部的 `%shared_out`，以纯数据流的形式表达内存的更新，维持了编译器所需的 SSA 语义。
- **计算切片与局部性优化 (Tiling)**：通过 `tensor.extract_slice`，各线程仅获取其负责的数据切片进行计算。此机制在编译期确立了访存边界，有利于优化底层硬件的缓存命中率。
- **并发拼接与隐式同步 (`in_parallel`)**：作为并发区域的终点屏障（Barrier）。内部配合 `tensor.parallel_insert_slice`，规范了各线程的局部结果应如何确定性地组合至全局张量中。
## 总结

| 特性   | Linalg Ops (Structured Ops)                     | Affine Loops                    | SCF Loops                  |
| ---- | ----------------------------------------------- | ------------------------------- | -------------------------- |
| 抽象层级 | 最高层 (算子级)                                       | 中层 (多面体/分析级)                    | 底层 (结构化控制流)                |
| 语义信息 | 包含迭代空间、数据访问 (Indexing Maps) 和计算逻辑 (Payload)     | 包含循环范围、步长，以及受限的下标表达式            | 仅包含基础的循环控制流 (lb, ub, step) |
| 优化优势 | Tiling, Fusion, Vectorization (通过 IR 结构直接保证合法性) | 依赖分析、Polyhedral 变换 (如交换、倾斜、并行化) | 基础控制流转换，是通往 LLVM IR 的桥梁    |
| 约束   | 限制在特定的线性代数算子模式                                  | 循环界限和访存下标必须是 Affine Expression  | 无特殊代数约束，支持动态边界             |

---
## 典型的 Lowering 路径

在实际的 MLIR 编译器（如 IREE 或 Torch-MLIR）中，一个张量循环经历的典型下沉路径为：
1. `Linalg` 阶段：进行算子融合（Fusion）和分块（Tiling）。
2. 转换 (Lowering)：
    1. 调用 `-convert-linalg-to-affine-loops` 下沉到 `Affine` 层进行深度的循环优化。
    2. 或者直接调用 `-convert-linalg-to-loops` 下沉到 `SCF` 层。
3. `SCF` 阶段：进行简单的循环剥离（Peeling）或展开（Unrolling）。
4. 最终阶段：从 `scf.for` 下沉到 `cf`，最后生成 LLVM IR。
---
## 参考

- [MLIR Linalg Dialect](https://mlir.llvm.org/docs/Dialects/Linalg/)
- [MLIR Affine Dialect](https://mlir.llvm.org/docs/Dialects/Affine/)
- [MLIR SCF Dialect](https://mlir.llvm.org/docs/Dialects/SCFDialect/)