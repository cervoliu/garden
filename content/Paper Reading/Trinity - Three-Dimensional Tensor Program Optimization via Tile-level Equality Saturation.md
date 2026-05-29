---
tags:
  - paper-reading
  - equality-saturation
  - tensor
feishushare: true
feishu_url: "https://feishu.cn/docx/N23QdiTfloVR6IxEoWpcSEclnKf"
feishu_shared_at: "2026-05-29 11:04"
---
## 论文总结

现代深度学习模型和硬件加速器在快速演化，然而现有的张量程序优化器在**图级别**（算子融合、代数重写）和**算子级别**（分块tiling、并行化）上各自独立运作，无法发现像 FlashAttention 那样需要跨层级联合推理的优化。Park 等人（2026）提出的 **Trinity** 是第一个通过 tile-level equality saturation 实现可扩展联合优化的张量程序优化器。 (Park et al. 2026, 1)

### 核心洞察：三个不可分割的优化维度

作者指出，最优性能要求同时优化三个相互依赖的维度：
- **代数等价性**：跨算子的代数变换（如分配律、结合律），能重构全局计算模式；
- **内存 I/O**：通过分块、缓存和重用模式管理数据在不同内存层级间的移动；
- **计算编排**：确定执行边界（哪些操作一起执行）和并行化策略（如何将工作分配到计算单元）。

这三者无法独立优化。代数变换改变计算和内存访问模式，内存布局影响并行化策略的可行性，并行化选择反过来决定哪种代数形式有利可图。 (Park et al. 2026, 2)

### 方法：Tile 粒度 IR + Equality Saturation

Trinity 设计了一个全新的细粒度中间表示（IR），以 **tile（瓦片）** 为基本粒度——即 GPU 上能放入片上缓存（scratchpad）的小块张量（如 64×64 元素）。在这个 IR 中：

- **内存操作是显式的**：`load`/`store` 是第一类可重写实体，使数据的加载、暂存、写出成为可优化的决策；
- **控制流是显式的**：`seq`（顺序执行）和 `loop`（迭代）作为可重写操作，使得核函数边界和循环结构可以被变换；
- **计算操作复用标准张量操作**：`matmul`、`+`、`*`、`exp`、`reduce_sum` 等，代数规则可以自然应用。 (Park et al. 2026, 4)

在这些基础上，Trinity 使用 **equality saturation**（等式饱和）来避免过早承诺某个优化策略：它将所有等价的程序表示压缩在一张 e-graph（等价图）中，通过反复应用重写规则扩展等价空间，最后通过成本模型从中提取最优程序。 (Park et al. 2026, 3–4)

### 两个关键技术挑战及解决方案

**挑战 1：状态化 IR 与等式饱和的冲突。** 传统等式饱和假设纯函数语义，但 Trinity 的 IR 包含显式的 load/store 和顺序控制流，这会阻碍代数规则的匹配（例如一个乘法和一个除法被中间的 store 和 load 隔开）。

解决方案：
- **表达式传播**：每遇到 `store`，Trinity 记录写入的符号表达式；后续的 `load` 被重写为直接引用该表达式，使得代数规则能够在显式内存操作之上正常工作。 (Park et al. 2026, 6)
- **序列规范化**：所有 `seq` 操作被强制保持在右结合规范形式，避免因结合律导致的 e-graph 指数膨胀。 (Park et al. 2026, 6)
- **语义依赖检查**：每条重写规则的触发都受 e-class 分析守护——规则仅在不会引入读后写（RAW）或写后写（WAW）冲突时才被应用。 (Park et al. 2026, 6)

**挑战 2：上下文依赖的成本模型。** 同一个算术 e-node（如 `(+ a b)`）在并行循环和顺序循环中的执行成本截然不同。

解决方案：**两阶段提取算法**。第一阶段提取出具有最少核函数数量的循环结构（因为核函数数量决定了 kernel launch 开销和核间内存流量）；第二阶段在已确定的循环结构下选择 FLOPs 最小的循环体。这使得在包含多达 10¹⁷ 个等价程序的 e-graph 中高效提取成为可能。 (Park et al. 2026, 7)

### 案例研究：全融合注意力机制

论文以 Vanilla Transformer 的解码步骤为例，展示了 Trinity 如何自动发现一种新颖的优化：

从一个朴素的 IR 程序出发（QKV 投影 + 多头注意力分属不同核函数），Trinity 依次应用：循环融合 + 分配律 → 代数因式分解打破循环携带依赖 → 进一步循环融合，自动复现了 **FlashAttention 的在线 softmax 算法**。更重要的是，Trinity 继续前进，通过迭代空间重索引实现循环融合，将 **QKV 投影、reshape/transpose、以及注意力计算全部合并到单个核函数中**——作者称这是首次被报告的"全融合注意力"优化。 (Park et al. 2026, 8–9)

这种全融合设计消除了等待所有 head 的 Q、K、V 计算完成的同步屏障，减少了 kernel launch 开销，并避免了中间张量（如 Q）写入全局内存。在 H100 上，Trinity 生成的核函数比手工优化的 FlashInfer 快 1.35 倍。 (Park et al. 2026, 9)

### 实验结果

Trinity 在多种 Transformer 变体（Vanilla、Pre-Norm、QK-Norm、KeyFormer、RoCo、SwiGLU FFN）和多种 GPU（H100、A100、RTX 4090、RTX 5090）上进行了评估：

- 相比 TensorRT（领先的生产级编译器），Trinity 在 H100 上实现最高 **2.09×** 的加速；相比 TorchInductor，最高 **2.35×**。 (Park et al. 2026, 1)
- 相比 Mirage（先前的联合优化器），Trinity 最高快 **3.07×**——因为 Mirage 的穷举搜索无法应对大规模搜索空间，不得不对程序进行分区优化，牺牲了跨分区优化机会。 (Park et al. 2026, 10)
- 编译时间合理：最复杂的架构 KeyFormer 的编译总时间约 1459 秒（含在多 GPU 上并行 profiling），而它探索的等价程序空间高达 10²¹。 (Park et al. 2026, 12)

一个特别有趣的发现是，Trinity 能自动适配不同硬件的特性：在 H100（高内存带宽 3 TB/s）上，它倾向于采用较大 tile（128）并适度使用 off-chip 内存溢写（spill）策略来提高吞吐；而在 RTX 4090（带宽仅 1 TB/s）上，它自动切换到更小的 tile（64），将中间结果保持在片上，避免带宽瓶颈。 (Park et al. 2026, 11)

### 局限与讨论

论文坦率讨论了几个限制：Trinity 当前仅处理规则的 tiling 模式（通过 `loop` 表示），不直接支持 Triton 级别的低层调优（如向量宽度、流水线阶段）；tile 大小在等式饱和阶段保持符号化，具体尺寸推迟到 profiling 阶段确定；代数重写遵循与 FlashAttention 等先前工作相同的浮点舍入假设，不是 bit-identical 的。 (Park et al. 2026, 8) (Park et al. 2026, 6) 此外，Trinity 目前聚焦于 Transformer 的推理阶段，扩展到训练和其他模型族（如状态空间模型、扩散模型）是未来的方向。 (Park et al. 2026, 9)

---
## 第 3 章：Trinity IR 详解

第 3 章是 Trinity 系统的基石——它定义了一种全新的中间表示（IR），使得三个优化维度（内存 I/O、计算编排、代数变换）能够在统一框架中被显式地重写和优化。
### 为什么要设计一个新的 IR？

现有张量编译器在两个层面分别优化：图级别（算子融合、代数重写）和算子级别（分块、并行化）。这两个层面之间通过**张量级算子**作为接口，但这恰恰是问题的根源——图级别看不到算子内部的分块和执行细节，算子级别看不到跨算子的代数变换机会。 (Park et al. 2026, 2)

Trinity 选择 **tile（瓦片）粒度**作为抽象层次。一个 tile 是张量的一个小块（如 64×64 元素），能放入片上缓存（scratchpad），在现代 GPU 上以 tile 为单位处理和调度。在这个粒度上：代数变换表现为可重排的一序列 tile 操作；内存 I/O 变为显式的 load/store；计算编排变为 loop 嵌套、seq 顺序、以及核函数边界的显式决策。 (Park et al. 2026, 3)

### 3.1 IR 定义：三大类构造

![[Constructs of Trinity IR.png]]

Table 1 列出了 Trinity IR 的全部构造，我把它们按用途分组讲解，并用具体例子说明。 (Park et al. 2026, 4)
#### 第一类：张量声明——区分内存位置

Trinity 用三种声明来刻画一个 tile 的"出身"：

- **`(input X)`**：输入张量，位于全局内存（off-chip），只读；
- **`(output X)`**：输出张量，最终要写回全局内存；
- **`(variable X)`**：中间张量，**可以放在片上也可以放在片外**——这个灵活性是发现复杂内存重用模式的关键。

举个具体例子：在 FlashAttention 中，softmax 的 running statistics（最大值、累加和）就是典型的 `variable`——它们希望在循环迭代间保留在片上 scratchpad 中以减少内存流量，但 IR 并不强制这个决定，而是留给优化器通过重写规则去探索。

#### 第二类：索引表达式——描述"哪一块"

当我们在一个循环中迭代 tile 时，需要描述当前迭代处理的是张量的哪个子块。Trinity 提供了三类索引：

- **`(tile n)`**：tile 切片 `[n : n + tile_n]`。`tile_n` 是循环变量 `n` 的步长（stride）。这是最常用的——"取循环变量 n 指示的那个 tile"。
- **`(full_tile)`**：完整切片 `[:]`，即取整个维度。用于不需要分块的维度（例如 batch 维度在推理时 batch=1 的情况）。
- **`(const_tile base interval)`**：常量切片 `[base : base + interval]`，与任何循环变量无关。用于在循环内始终访问同一个固定 tile。
- **`(elem n)`**：元素级索引 `[⌊n / tile_n⌋]`。注意它与 `(tile n)` 的区别——`tile` 给出的是一个块，而 `elem` 给出的是 tile 序号本身（用于跨循环的索引重对齐）。

这四种索引的组合使得 Trinity 既能描述规则的 tiling 模式，又能在循环融合时通过重索引实现迭代空间的对齐。§5.3 的全融合注意力案例中，最后一步正是利用了 `(elem n)` 和 `(elem h)` 的等价性来统一两个看似不同步长的循环。 (Park et al. 2026, 9)

#### 第三类：内存访问操作——显式的 load/store

这是 Trinity IR 与传统纯函数式 IR 最根本的区别：

- **`(load tensor idx)`**：从张量的 idx 位置加载一个 tile 值；
- **`(store tensor tval idx)`**：把 tile 值 `tval` 存到张量的 idx 位置。

在 TensorRT 或 TorchInductor 的图 IR 中，你看到的是 `MatMul(A, B)` 这样抽象的算子，内存读写是隐式的。在 Trinity IR 中，**load 和 store 是第一类可重写实体**。这意味着优化器可以直接做这样的变换：

> 如果两次 `(load A idx)` 之间没有对 A 的 `store`，则第二次 load 可以消除，直接复用第一次 load 的结果。

这正是手工优化的核函数中常见的"把频繁访问的 tile 保持在寄存器/scratchpad"的逻辑，但 Trinity 把它变成了等式饱和框架下的重写规则，可以自动发现和应用。

#### 第四类：计算操作——tile 上的代数

Trinity 把 tile 视为"小张量"，直接复用标准张量计算操作：

- 逐元素运算：`(+ tval1 tval2)`、`(- tval1 tval2)`、`(* tval1 tval2)`、`(/ tval1 tval2)`
- 单元运算：`(exp tval)`、`(sqrt tval)`、`(sigmoid tval)`
- 归约：`(rsum tval k)`——沿第 k 轴做 reduce sum
- 矩阵乘：`(matmul tval1 tval2)`
- 形状变换：`(concat ...)`、`(permute ...)`、`(unsqueeze ...)`、`(squeeze ...)`、`(bcast ...)`

由于这些操作的语义与标准张量库一致，代数规则（分配律、结合律、交换律）可以直接应用于 tile 级别。这保证了"图级代数重写"的能力被完整保留在了更细粒度的 IR 中。 (Park et al. 2026, 5)

#### 第五类：控制流操作——可重写的执行顺序

这是 Trinity IR 的第二个关键创新——将控制流也变成可重写实体：

- **`(seq op1 op2)`**：依次执行 `op1` 和 `op2`。**seq 不是语法糖——它是可被交换、可被重排的 IR 节点。**
- **`(loop start end tile_n n op)`**：以 `n` 为循环变量，从 `start` 到 `end`（不含），步长为 `tile_n`，执行循环体 `op`。

`loop` 的语义需要仔细理解。例如：

```
(loop 0 4096 128 n
 (matmul (load Q (tile n)) (load K (full_tile))))
```

这表示：将 Q 按 128 为步长分块（共 4096/128 = 32 次迭代），每次迭代取 Q 的一个 tile 与完整的 K 做矩阵乘。注意 `(tile n)` 给出了一个 [n:n+128] 的切片。

**`seq` 和 `loop` 为什么必须是可重写的？** 考虑两个相邻的 loop：

```
(seq
  (loop 0 4096 128 n (compute_QKV ...))
  (loop 0 32 1 h (attention ...)))
```

如果这两个循环之间没有依赖，循环融合规则可以将它们合并：

```
(loop 0 32 1 h
  (seq (compute_QKV ...) (attention ...)))
```

这样 QKV 投影和注意力计算就进入了**同一个核函数**——这正是全融合注意力的关键一步。如果 `seq` 和 `loop` 是不可重写的语法糖，这种跨算子边界的优化就无从谈起。 (Park et al. 2026, 5)

---

### 一个完整示例：矩阵乘 + 逐元素加法

论文的 Figure 3（第 5 页）展示了一个具体的 IR 程序，表示两个二维矩阵相乘后再做逐元素加法：

```
(loop 0 M tile_m m
  (loop 0 N tile_n n
    (loop 0 K tile_k k
      (store C (+ (load C (index m n))
                  (* (load A (index m k))
                     (load B (index k n))))
             (index m n)))))
```

这里可以看到：
- **tiling** 沿三个维度展开：输出行 M、输出列 N、归约轴 K；
- **显式 load/store**：A、B、C 的每个 tile 都需要显式加载，结果显式存回 C；
- **归约通过逐步累加实现**：`(+ (load C ...) (* (load A ...) (load B ...)))`——每次迭代把 A[m,k]×B[k,n] 累加到 C[m,n] 上。

这个初始程序虽然正确，但远非最优。Trinity 的等式饱和框架会在此基础上不断应用重写规则，自动探索等价但更高效的执行方式。

---

### 3.2 重写规则：两大类别

Trinity 的重写规则分为循环变换和代数变换两大类，全部列在附录 A 中。 (Park et al. 2026, 17–19)

#### 循环变换规则

这些规则操作 `loop` 和 `seq` 结构，改变核函数边界和计算编排。每条规则都有**安全性条件**——通过 e-class 分析检查读写依赖，确保不引入 RAW（读后写）或 WAW（写后写）冲突。

**循环融合（loop-fusion / loop-fusion-tail）**是最核心的规则。两个具有相同迭代空间的相邻循环，如果它们之间没有依赖冲突，就可以合并为一个循环：

```
(seq (loop n tile_n var body1)
     (loop n tile_n var body2))
  → (loop n tile_n var (seq body1 body2))
```

融合后消除了一次 kernel launch 的开销，而且可以让 body1 产生的中间 tile 直接保留在片上被 body2 使用，无需写回全局内存。

**循环裂变（loop-fission / loop-fission-tail）**是融合的逆操作——把一个循环拆成两个。为什么需要裂变？因为不同的循环体可能需要不同的并行化策略（不同的 grid/block 配置），拆分后可以各自获得最优的硬件调度。

**循环插入（loop-insertion）**是 Trinity 最具创新性的规则之一。传统编译器做"循环不变量外提"（loop-invariant code motion）来消除冗余计算，但 Trinity 在推理场景下**故意把循环外代码插入循环内部**：

```
(seq (loop 0 n tile_n var body1) body2)
  → (loop 0 n tile_n var (seq body1 body2))
```

虽然 `body2` 会被重复执行（增加了计算量），但它现在和 `body1` 在同一个循环内，可以融合进同一个核函数。在推理场景中，kernel launch 开销和内存 I/O 往往远大于计算开销，因此这种 trade-off 是值得的。§5.3 的 Pre-Norm 优化中，RMS 归一化计算正是通过循环插入被合并到 QKV 投影循环中的。

**循环体代数因式分解（alge-factor-loop-body）**是另一个巧妙的设计。以 FlashAttention 为例，输出 O 的计算包含 `logit * V / accm`，其中 `accm` 是在内层循环中累加得到的。这条规则将除以 `accm` 的操作提到循环外面：

```
(loop p ... (store b (+ (* (load b) accm) (/ val1 val2))))
  → (seq (loop p ... (store b (+ (* (load b) accm) val1)))
         (store b (/ (load b) val2)))
```

一旦 `val2`（即 `accm`）不再出现在内层循环中，循环携带的依赖就解除了，为进一步的循环融合打开了大门。这正是图 4 中从步骤 (b) 到 (c) 的关键变换。 (Park et al. 2026, 8–9) (Park et al. 2026, 18)

#### 代数变换规则

Trinity 包含 31 条代数重写规则，沿用了先前工作（如 TASO、FlashTensor）中针对张量级别的代数变换，但直接应用于 tile 粒度。 (Park et al. 2026, 19)

这些规则可以分为几组：

**基本代数律**：交换律 `(+ a b) → (+ b a)`、结合律 `(+ a (+ b c)) → (+ (+ a b) c)`、分配律 `(* a (+ b c)) → (+ (* a b) (* a c))`，以及它们对减法和矩阵乘的版本。分配律在 FlashAttention 推导中至关重要——它将 `logit * V / accm` 重写为 `logit * V * (1/accm)`，从而解除了跨循环迭代的依赖。

**矩阵乘与 concat 的交互**：例如 `(matmul x (concat y z 1)) → (concat (matmul x y) (matmul x z) 1)`。这些规则允许优化器在"先拼接再乘"和"先乘再拼接"之间选择更高效的方案。

**exp 相关的规则**：`(* (exp a) (exp b)) → (exp (+ a b))` 和 `(/ (exp a) (exp b)) → (exp (- a b))`。在 softmax 计算中，这些规则使得在线 softmax（online softmax）的代数重写成为可能——将全局 softmax 分解为逐 tile 的局部 softmax 与 running statistics 的组合。

**concat 几何变换**：`(concat (concat x z 0) (concat y w 0) 1) → (concat (concat x y 1) (concat z w 1) 0)`，类似于矩阵转置中的分块互换。这在多头注意力的 reshape/permute/transpose 操作序列中可能触发新的融合机会。

---

### IR 设计的深层考量

Trinity IR 的精妙之处在于它同时满足了三个看似冲突的要求：

**足够细以暴露硬件细节**：load/store/loop/seq 全是显式的，核函数边界、内存放置、并行化策略都可以被重写规则探索。这不同于 TensorRT 的图 IR（算子内部不透明）和 Mirage 的 IR（内存 I/O 隐含在模板中）。

**足够粗以避免组合爆炸**：IR 刻意只覆盖规则的 tiling 模式，不引入低层级的语法细节（如 Triton 中的向量宽度、流水线阶段数）。tile 大小在等式饱和阶段保持符号化，具体数值推迟到 profiling 阶段确定。这避免了 e-graph 的指数膨胀。 (Park et al. 2026, 3) (Park et al. 2026, 8)

**保持代数可组合性**：因为 tile 被视为"小张量"，所有张量级别的代数规则（分配律、结合律等）可以直接应用于 tile 操作，无需修改。loop 变换和代数变换在统一的 e-graph 框架中交织进行，这正是"三维联合优化"的机制基础。
