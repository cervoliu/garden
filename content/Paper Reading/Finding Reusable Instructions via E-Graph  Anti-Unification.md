---
tags:
  - paper-reading
  - ml-compiler
  - codegen
  - e-graph
  - anti-unification
---
## 研究背景与动机

现代计算领域（如数字信号处理、人工智能）对算力的需求已超越通用处理器的能力。**自定义指令**（Custom Instructions, CIs）配合专用硬件单元成为平衡性能与成本的关键技术，而 RISC-V 开放可扩展的指令集架构进一步推动了这一趋势。然而，现有自动化方法存在两大缺陷：**复用性差**（仅基于语法相似性进行合并，忽略语义等价）和**缺乏并行性**（无法生成向量化自定义指令）。

论文给出了一个具体例子说明这个问题：对于 CImg 图像处理库，基于语法合并的方法生成了一条庞大且过度特化的自定义指令，仅被 8 处代码使用；而考虑语义复用性后，每条自定义指令平均可加速 93 处代码，在节省 90.5% 面积的同时获得 1.17× 的额外加速 (Xiao et al. 2026, 2)。

## 核心方法：RII（可复用指令识别）

ISAMORE 的核心理念是通过 **e-graph 反合一**（anti-unification, AU）从代表性应用中抽象出共同的语义模式。e-graph 是一种紧凑表示语义等价项的数据结构。通过等式饱和（equality saturation, EqSat）施加等价重写规则后，RII 对 e-class 对进行反合一，识别出至少出现两次的通用模式，从根本上保证了自定义指令的复用性。

以论文图 1 为例：传统方法对 `a×2+b×2` 和 `(1+i)<<1` 进行语法合并，生成包含四个操作和三个多路选择器（MUX）的低效指令。而 ISAMORE 通过语义感知的方法，识别出简洁的模式 `(x+y)×2`，只需两个操作即可复用  (Xiao et al. 2026, 3)。

## ISAMORE 框架

ISAMORE 是一个端到端框架，工作流程如下  (Xiao et al. 2026, 4)：

1. **结构化 DSL**：将 LLVM IR 翻译为结构化领域特定语言，能表示程序的控制流（If、Loop）和数据流，以及在 e-graph 中进行编码。
2. **RII 核心流程**：分阶段迭代进行模式识别。
3. **硬件感知选择**：基于性能分析和 HLS（高层次综合）的成本模型，进行帕累托最优的模式选择。
4. **指令生成**：将选定的模式翻译为自定义指令并生成 Verilog 硬件实现。

## 关键技术贡献

### 1. 分阶段迭代（Phase-Oriented Iteration）

直接用全部重写规则做单次 EqSat 会导致 e-graph 规模爆炸，LLMT 原始方法在超过 150 个 e-class 时就会内存溢出（>30GB），而实际应用通常超过 2000 个 e-class。RII 的解决策略是：每个阶段只应用一小部分精心选择的重写规则集，先饱和处理 sat 规则集，再逐步引入非饱和规则，在等价性探索和 e-graph 规模之间取得平衡  (Xiao et al. 2026, 4) (Xiao et al. 2026, 6)。

### 2. 智能 AU（Smart AU）

为解决 e-graph AU 的指数级复杂度，RII 引入两项启发式技术：

- **基于相似度的 e-class 配对**：利用结构哈希（structural hashing）和类型系统过滤，只对结构相似且类型一致的 e-class 对进行 AU，而非穷举所有配对。
- **启发式模式采样**：对每对 e-node 产生的 AU 模式，通过 boundary 策略（仅保留极值模式）或 kd-tree 策略（在特征空间中均匀采样）来避免模式数量的指数爆炸。

以论文图 8b 为例：对于 e-class ⑨ 和 ⑩，结构哈希计算显示它们相似度超过阈值，于是配对进行 AU；而 ⑨ 和 ⑦ 的相似度不足，直接跳过  (Xiao et al. 2026, 7)。

### 3. AU 驱动的模式向量化（Pattern Vectorization）

RII 从标量程序中挖掘数据级并行（DLP）机会，核心步骤包括  (Xiao et al. 2026, 7–8)：

- **种子打包**：在同一基本块内识别属于相同模式的多个实例，用 Vec 节点统一。
- **打包扩展**：通过 lift/couple 重写规则，恢复向量构造器，构建标量-向量混合 e-graph。
- **无环剪枝**：消除 Get→Vec→Get 循环，生成轻量级无环 e-graph 用于后续模式识别。

### 4. 硬件感知的多目标选择

RII 的代价模型基于 GEM5 性能分析和 XLS HLS 引擎的硬件估计，计算每条模式的延迟节省（ΔL）和面积开销（A），通过 e-class 分析传播帕累托前沿，在加速比和面积之间探索最优权衡  (Xiao et al. 2026, 8)。

## 实验结果

论文在 9 个基准核（包括 2DConv、MatMul、FFT、Stencil、SHA 等）及一个复合基准 All 上评估，主要结果如下  (Xiao et al. 2026, 9–11)：

**RII 可扩展性**：RII 将峰值 e-graph 规模降低 6–39×，所有基准在 145 秒内完成，内存不超过 799MB；而原始 LLMT 方法在所有案例上内存溢出。

**性能对比**：
- 相比 NOVIA（粗粒度语法合并），ISAMORE 最大加速方案平均加速 1.52×（最高 1.94×）。
- 相比 ENUM（细粒度枚举），ISAMORE 在相似加速比下面积开销更小。
- 相比 NoEqSat（跳过 EqSat），ISAMORE 平均快 1.12× 且面积仅为其 84.9%。
- 启用向量化后，性能优势升至平均 1.76×（最高 2.69×）。

**真实库分析**：在 liquid-dsp、CImg、PCL 三个领域的开源库上，ISAMORE 相比 NOVIA 加速 1.17×–2.73×，同时节省 84%–93% 的面积。

**LLM 推理案例**：对 BitNet b1.58 的推理，ISAMORE 识别出低位点积的向量化模式，在 Rocket 处理器上实现 2.15× 加速，仅增加 4.81% 面积。

**后量子密码学案例**：对 CRYSTALS-KYBER，ISAMORE 识别出蝶形运算作为可复用自定义指令，实现 5.15× 加速。

## Sources

Xiao, Youwei, Chenyun Yin, Yitian Sun, Yuyang Zou, and Yun Liang. 2026. “Finding Reusable Instructions via E-Graph Anti-Unification.” Proceedings of the 31st ACM International Conference on Architectural Support for Programming Languages and Operating Systems, Volume 2 (New York, NY, USA), ASPLOS ’26, 749–63. https://doi.org/10.1145/3779212.3790162.