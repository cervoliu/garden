---
tags:
  - paper-reading
  - tensor
  - ml-compiler
feishushare: true
feishu_url: "https://feishu.cn/docx/R8lmdvUTLoGKpGxWvAkcyTMYnwG"
feishu_shared_at: "2026-05-26 13:14"
---
### 1. 问题与动机

近年来，深度学习框架（如 TensorFlow、PyTorch）和专用硬件加速器（如 Google TPU、Intel Gaudi）极大地改变了计算密集型代码的编写和执行方式。然而，每种框架和硬件都提供自己的一套 API/ISA（即领域特定语言，DSL），开发者必须**为每个目标平台重新编写代码**。随着新框架和加速器的不断涌现，现有代码很快变成遗留代码，手动迁移成本高昂且容易引入错误 (Qiu et al. 2024, 1–2)。

传统方案是构建转译器（transpiler），例如 Dexter、STNG、C2TACO 等。但这类转译器通常是针对单一目标 DSL 定制的，难以扩展到新的运算符或后端。大型语言模型（LLM）虽在代码翻译方面展现出潜力，但**无法保证输出代码的正确性**，对于训练数据中未出现的新 DSL 几乎无法生成语法正确的代码  (Qiu et al. 2024, 2)。

### 2. 核心方案：验证式提升（Verified Lifting）+ TensIR

**Tenspiler** 的核心思想是用**程序合成（program synthesis）** 替代传统编译器的模式匹配方法，将通用语言（C++ / Python）编写的顺序程序自动翻译为等价的张量运算。它使用一种称为**验证式提升**的技术：通过归纳程序合成，推断出与源程序可证明等价的程序摘要（program summary），然后用该摘要生成目标代码  (Qiu et al. 2024, 2–3)。

Tenspiler 的关键创新是设计了一个小而精简的中间表示语言——**TensIR**。TensIR 基于张量代数，包含了常见的向量和矩阵运算（如逐元素运算、张量-向量乘法、规约操作等），其精妙之处在于：**只抽象出各目标 DSL 共有运算符的语义，而忽略低层实现细节**（如 tiling、向量化）。这使得同一个 TensIR 程序可以轻松翻译到多种后端  (Qiu et al. 2024, 2–4)。

### 3. 一个贯穿全文的例子

论文以一个**图像混合（blend）** 操作为例贯穿全文。一段 C++ 代码包含双重嵌套循环，对每个像素执行 `a + b - (a * b) / 255` 的计算。Tenspiler 做三件事：

1. **合成阶段**：将这段循环代码提升为 TensIR 表达式：`t_t(t_t(b, a, +), t_s(t_t(b, a, *), 255, /), -)`，其中 `t_t` 表示逐元素二元张量运算，`t_s` 表示逐元素标量-张量运算。同时合成循环不变量（loop invariants）来帮助验证  (Qiu et al. 2024, 4–5)。

2. **验证阶段**：用 SMT 求解器 CVC5 对**无界域**（所有可能的程序状态）进行验证，确保合成程序与源程序在所有输入下等价  (Qiu et al. 2024, 9)。

3. **代码生成阶段**：通过简单的模式匹配规则，将 TensIR 表达式翻译为目标 DSL 的具体语法——例如 NumPy 中变为 `b + a - b * a // 255`，Gaudi 的 TPC-C 中变为一系列向量化 intrinsic 指令  (Qiu et al. 2024, 4)。

### 4. TensIR 的设计

TensIR 的语法非常紧凑（如图 2 所示），包括以下几类运算符：

- **逐元素运算**：`tensor_scalar`（张量与标量）和 `tensor_tensor`（张量与张量），支持加减乘除取模。
- **张量操作**：`transpose`（转置）、`tensor_vec_prod`（张量-向量乘积）。
- **规约运算**：`reduce_max`、`reduce_sum`。
- **控制流**：`ite`（if-then-else），用于翻译带分支的循环代码。
- **访问操作**：`take`、`tail`、`slice`、`size`。

TensIR 的设计与 MLIR 形成鲜明对比：MLIR 的现有 dialect（如 linalg 和 tensor）无法覆盖所有需要的运算（例如 `select` 运算符在图像处理中十分关键但不受 MLIR 原生支持），且学习曲线较陡；而 TensIR 开发者只需描述运算符的高层语义并定义简单的模式匹配规则，即可添加新运算符或新后端  (Qiu et al. 2024, 5–7)。

### 5. 合成优化：让搜索可扩展

这是论文最重要的技术贡献之一。直接枚举 TensIR 所有可能的表达式（深度为 4）会产生约 20 万个候选项，搜索无法规模化。Tenspiler 设计了四项关键优化  (Qiu et al. 2024, 11–13)：

1. **基于类型的运算符限制**：根据返回类型排除不匹配的运算符（例如返回类型是二维张量时，排除所有规约运算）。

2. **限制程序状态（Bounded Synthesis）**：初始合成时将张量长度限制为 2、整数限制为 6 位，大幅缩小搜索空间。合成后由定理证明器验证无界正确性，失败则增大界限。

3. **表达式树引导（核心优化）**：对源程序进行静态分析，提取计算表达式树，将其转化为抽象模板（将变量和常量替换为占位符）。例如从 `(- (+ b a) (/ (* b a) 255))` 推导出模板 `t_t(t_t(var, var, +), t_s(t_t(var, var, *), lit, /), -)`，然后只需合成 `var` 和 `lit` 的具体值。这项优化将搜索空间从 ~10 万个表达降至 64 个，在 76 秒内完成合成。

4. **变量与常量约束**：将变量限制为活跃变量集合，常量限制为程序中出现的常量集合。

消融实验表明，没有表达式树优化时，42/69 个基准测试超时；没有增量式有界合成时，6/12 个 blend 基准测试超时  (Qiu et al. 2024, 21–22)。

### 6. 实验评估

Tenspiler 在 **10 个真实代码基准套件**上进行了全面评估，涵盖图像处理（blend）、深度学习推理（Llama）、线性代数（BLAS）、信号处理（DSP）等领域，共计 69 个函数。支持 6 种目标 DSL：NumPy、TensorFlow、PyTorch、MLX（Apple 芯片）、TPC-C（Intel Gaudi）、Gemmini  (Qiu et al. 2024, 13–15)。

**合成时间**：所有基准测试均在 15 分钟内完成合成，大多数在秒级。例如，单循环的 `dot` 约需 2 秒；双循环的 `transformer_part1` 约需 1300 秒（因为需要合成 6 个运算符及其所有参数）。合成难度与循环数量和 TensIR 方案的复杂度正相关  (Qiu et al. 2024, 16–17)。

**性能提升**：
- **Kernel 性能**：平均 105 倍加速（Gaudi 2 达到 241 倍），TensorFlow 46 倍，PyTorch 244 倍  (Qiu et al. 2024, 17–19)。
- **端到端性能**：平均 9.65 倍加速（数据传输开销会抵消部分 kernel 收益） (Qiu et al. 2024, 19)。
- **与 Numba 对比**：Tenspiler 生成的 PyTorch/TensorFlow 代码平均比 Numba 标注的 CUDA 代码快 1.87 倍。原因是 Numba 生成的 PTX 汇编缺乏 FMA（融合乘加）、tiling 和共享内存等高级优化技术  (Qiu et al. 2024, 19–21)。

**与 LLM 对比**：用 Claude Opus 为 MLX、Gaudi、Gemmini 生成代码——三个输出全部错误（幻觉 TPC-C 库、错误的 API 调用、错误的导入），且无法形式化验证其正确性  (Qiu et al. 2024, 23)。

**新后端的易扩展性**：为 MLX（发布仅四个月的新框架）添加支持仅需不到 200 行代码  (Qiu et al. 2024, 10)。

### 7. 局限与未来方向

- Tenspiler 目前仅支持 C/C++ 和 Python 的子集（不支持指针和对象），且需要对外部库进行功能建模。
- 仅处理 1D 和 2D 张量（但论文展示了可扩展到 3D 的可行性实验）。
- 合成和验证依赖 SMT 求解器对整数和实数的推理，浮点数的验证仍受限。
- 未来方向包括：用机器学习引导搜索过程、从内层循环开始的 bottom-up 合成、基于展开循环的有界合成  (Qiu et al. 2024, 24)。

### 小结

Tenspiler 的核心洞察是：设计一个既足够表达多种张量运算 DSL 的公共语义、又足够简洁使程序合成可扩展的中间表示（TensIR），从而用一个统一的 verified-lifting 框架替代传统的、每种 DSL 各自实现的模式匹配编译器。它的实用价值体现在：69 个真实世界的循环程序自动翻译到 6 个不同的软硬件平台，kernel 平均加速 105 倍，且新后端仅需 ~200 行代码即可接入。

## Sources

Qiu, Jie, Colin Cai, Sahil Bhatia, Niranjan Hasabnis, Sanjit A. Seshia, and Alvin Cheung. 2024. “Tenspiler: A Verified Lifting-Based Compiler for Tensor Operations (Extended Version).” arXiv:2404.18249. Preprint, arXiv, December 14. https://doi.org/10.48550/arXiv.2404.18249.