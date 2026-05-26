---
tags:
  - paper-reading
  - translation-validation
  - OOPSLA25
feishushare: true
feishu_url: "https://feishu.cn/docx/YB2td3cOJoxZFtxZtmpcnF7Jnrg"
feishu_shared_at: "2026-05-26 23:01"
---
### 动机与背景

现代编译器后端不仅仅做寄存器分配和指令选择，它本身就是一个高度优化的编译器。LLVM 的后端有大约一百万行 C++ 代码，包含数据流分析、公共子表达式消除、循环不变量代码移动等大量优化 pass，并且有自己的中间表示 MIR（Machine IR）。这些复杂的优化带来了巨大的正确性风险。对于安全关键型软件（如汽车领域），目前的"最佳实践"是在整个产品生命周期中锁定一个编译器版本——不是因为它正确，而是因为它的缺陷和规避方法已经为人所知。作者团队的目标是创造一种方法，让工程师可以信任 LLVM 的新版本。 (Berger et al. 2025, 3)

### arm-tv 是什么

arm-tv 是一个 **translation validation**（翻译验证）工具，它形式化地验证 LLVM IR 到 AArch64 汇编代码的单次翻译是否正确。它的核心思路是：将 LLVM 后端生成的 AArch64 汇编代码 **lift（提升）** 回 LLVM IR，然后使用 Alive2（一个已有的 LLVM 验证后端）检查提升后的 IR 是否 **refine（精化）** 原始的 LLVM IR。 (Berger et al. 2025, 6)

所谓 refinement，是指转换后的程序的行为是原始程序行为的子集——这是编译器正确性的标准准则。关键性质是 refinement 具有组合性：如果编译器执行的每一个单独变换都是 refinement，那么整个编译过程也是 refinement。 (Berger et al. 2025, 5)

### 核心技术贡献

**1. 双轨制指令语义实现。** arm-tv 提供了两种为 AArch64 指令赋予语义的方式。第一种是手工编写，作者团队根据 ARM ISA 参考手册手工实现了约 1,400 条指令的语义（约 8,100 行 C++）。第二种是利用 ARM 官方的机器可读架构（MRA），通过改进的 ASLp 部分求值器，从 ASL（Architecture Specification Language）自动推导出指令语义（约 2,000 行 C++）。这是首次对手工编写和机器推导的 ISA 语义进行同类对比。 (Berger et al. 2025, 11) (Berger et al. 2025, 20)

**2. 为 ASLp 添加向量化支持。** ASL 规范将向量指令描述为循环（逐元素操作），ASLp 最初会展开这些循环，导致代码标量化，验证性能极差。作者为 ASLp 增加了约 1,000 行 OCaml 代码的向量化 pass，使其能生成原生的向量操作，显著缩小了与手工编写 lifter 之间的性能差距。 (Berger et al. 2025, 13)

**3. 汇编级内存模型。** 这是最重要的贡献之一。LLVM IR 中的指针带有 provenance（来源信息），类似于 capability，只能访问特定的内存对象。但汇编层面的指针就是普通的 64 位整数，可以访问整个地址空间。两者之间存在根本性的语义不匹配。作者在 Alive2 中新增了一个"汇编模式"，改变了指针语义（全部视为 physical pointer，拥有 full provenance），并实现了两项关键的内存编码优化：**synchronizing alignment** 和 **inbounds alignment**，将汇编模式下的验证性能提升了近一个数量级。对于 LLVM 单元测试，超时率从 7.0% 降至 3.1%。 (Berger et al. 2025, 15–17) (Berger et al. 2025, 20)

**4. ABI 规则检查。** arm-tv 不仅检查功能正确性，还检查大量 ABI 层面的属性：栈指针是否 16 字节对齐、被调用者保存寄存器是否恢复、signext/zeroext 属性是否被遵守等等。这是区分 arm-tv 与以往工作的一个重要特点。 (Berger et al. 2025, 10–11)

**5. 处理 lifting 中的语义失配。** 将汇编提升回 LLVM IR 需要极为小心地保持 refinement。例如 LLVM 的 `sdiv` 指令在除数为零时有未定义行为，但 AArch64 的 `sdiv` 在这种情况下返回零。作者的处理方法是：在提升代码中显式检查会触发未定义行为的输入条件，然后才执行 LLVM 指令，这导致提升后的代码可能比原始的汇编代码长很多（如论文中 Fig. 1 所示，一个简单的除法函数需要 9 条 LLVM 指令来正确表达）。 (Berger et al. 2025, 9–10)

### 评估与发现

作者在两个测试集上评估了 arm-tv：LLVM 单元测试套件（232,981 个函数）和 SPEC CPU 2017（87,738 个函数）。

对于 LLVM 单元测试，arm-tv 能成功验证绝大多数小到中型函数，最常见的阻塞因素是向量参数类型缺乏稳定的 ABI（占未支持函数的 55%）。SPEC 代码更难验证，主要是因为它大量使用指针、函数指针和多级间接引用，超时率约为 40%。 (Berger et al. 2025, 19)

在 bug 发现方面，作者使用 arm-tv 结合 YARPgen、alive-mutate 和 IRFuzzer 等模糊测试工具，在 2022 年 4 月至 2025 年 7 月期间发现并报告了 **45 个此前未知的 LLVM 错误编译 bug**，其中 39 个已被修复。这些 bug 涉及整数运算、浮点运算、向量操作、内存访问、未定义行为处理等多个类别，许多 bug 影响的代码被多个后端共享。 (Berger et al. 2025, 22–23)

论文中给出了两个有代表性的 bug 实例。第一个（#74248）涉及 `insertelement` 指令：当向量索引超出范围时，LLVM IR 规定结果是 poison 值，但后端将其翻译成了一条会覆盖任意内存的单字节存储指令。第二个（#57181）涉及 ABI 中布尔参数扩展规则的歧义：LLVM 的 signext 属性要求符号扩展到 32 位，而 ARM ABI 规定布尔参数零扩展到 8 位，两个后端的处理方式不一致。 (Berger et al. 2025, 5–6) (Berger et al. 2025, 23–24)

### 局限与未来工作

arm-tv 目前只支持顺序用户态代码，不支持特权指令、内联汇编、线程局部存储、变参函数、非标准浮点类型等。它继承了 Alive2 的限制，包括不支持无界循环和异常处理。对于大型函数（特别是大量内联产生的），验证可扩展性仍然是一个问题。作者正在为 64 位 RISC-V 后端添加支持，目前已支持 RV64I 以及 B 和 M 扩展。 (Berger et al. 2025, 18) (Berger et al. 2025, 25)

### 总结

arm-tv 是一个工程上实用、学术上创新的 translation validation 工具，它不仅证明了 LLVM AArch64 后端中存在大量此前未被发现的 bug，还通过汇编级内存模型和 ASLp 向量化等技术创新，推动了整个 translation validation 领域向前发展。