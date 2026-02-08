---
tags:
  - MLIR
  - paper-reading
---
## 总结

这篇论文提出了一种将形式化语义作为 MLIR（Multi-Level Intermediate Representation）生态系统一等公民的方法。MLIR是一个用于构建可扩展和可组合的中间表示（IRs）的工具包，但其主要限制在于它是以语法为中心的，缺乏直接编码操作语义的能力。这导致优化器、分析器、验证器和转换器等依赖语义的工具需要手工实现语义逻辑。

**核心问题与解决方案：**
论文的核心贡献是设计并实现了一系列支持语义的 MLIR 方言（"semantic dialects"），用于编码编译器IRs的语义。这种方法实现了在构建基于形式化方法的编译器工具时，不同专业领域之间的关注点分离：
1.  **编译器开发者**：通过将他们的方言“降级”（lowering）到论文提出的语义方言之一或多个，来定义其方言的语义。这使得开发者可以使用他们熟悉的编译器转换技术来表达语义。
2.  **SMT求解器专家**：提供工具来优化领域特定的高级语义，并将其降级为SMT查询。
3.  **工具构建者**：创建与方言无关的验证工具。

**技术细节：语义方言的层级结构**
论文引入的语义方言体系分为两大部分：低级SMT-LIB方言和高级语义方言。

1.  **SMT-LIB方言 (`smt`, `smt_int`, `smt_bv`, 等)**：
    *   这些是最低层的方言，直接对应SMT-LIB v2语言中的概念。每个SMT-LIB符号都由一个MLIR操作表示，SMT-LIB的“sorts”（类型）转换为MLIR类型。
    *   例如，`declare-const x Int`在MLIR中表示为`%x = smt.declare_const : !smt_int.int`。
    *   这种直接映射使得从MLIR程序生成SMT-LIB查询变得容易。核心`smt`方言及其理论方言（如`smt_int`和`smt_bv`）已被贡献给上游MLIR项目。

2.  **高级语义方言 (`poison`, `effect`, `ub_effect`, `mem_effect`, `memory`)**：
    *   为了弥合编译器开发者定义的高级语义与SMT-LIB之间的抽象鸿沟，论文定义了一组更高级的语义方言。
    *   **`poison`方言**：用于显式标记值是否为“poison”（毒值），对应MLIR中延迟的未定义行为。`!poison.poison<T>`类型表示一个包含值 `T` 和一个布尔毒值标记的对。相关的操作包括`poison.from_value`、`poison.poison`、`poison.is_poison`和`poison.to_value`。
    *   **`effect`方言**：定义`!effect.state`类型，用于表示全局状态（如内存和未定义行为标志）。
    *   **`ub_effect`方言**：用于表示未定义行为。`ub_effect.trigger(s)`将状态 `s` 转换为未定义状态 `ub`。`ub_effect.to_bool(s)`判断状态是否为未定义。
    *   **`mem_effect`方言**：用于表示顺序内存访问语义，基于Lee等人[20]的工作。它定义了诸如指针操作(`mem_effect.offset_ptr`)、内存读写(`mem_effect.read`, `mem_effect.write`)和内存分配(`mem_effect.alloc`)等高级概念。
    *   **`memory`方言**：提供更低级的内存操作抽象，如字节读写(`memory.read_bytes`, `memory.write_bytes`)、块管理(`memory.create_live_block`, `memory.get_block`)和块ID管理(`memory.get_fresh_block_id`)。
    *   通过这些高级方言，编译器开发者可以以更接近其心智模型的方式描述语义，而SMT专家则可以通过专门的编译passes将这些高级语义高效编码为SMT查询。例如，`arith.addi`操作的语义可以降级为包含毒值检查和 `smt_bv.add` 操作的语义表示。

**语义定义与优化：**
*   **通过编译器转换定义语义**：为MLIR方言定义语义，实际上就是定义一个从该方言到语义方言的降级（compilation pass）。这与MLIR中通过降级来定义源方言语义的现有哲学保持一致。
*   **多层级SMT查询优化**：由于语义在多个抽象层级上进行编码，编译器专家和SMT专家可以合作在每个层级定义领域特定的优化。这包括：
    *   **常规优化**：每个语义方言都定义了常量折叠和简单的窥孔重写规则，以减少SMT编码的大小。
    *   **公共子表达式消除**：进一步减少SMT查询中的操作数量。
    *   **领域特定优化**：例如，一个编译pass可以移除SMT查询中的代数数据类型（`smt.tuple`），因为某些SMT求解器（如Z3）在处理这些高级抽象时性能不佳。论文实验显示，这种优化显著提升了查询性能（加速比达24.6%到59.5%）。
    *   **内存语义编码**：将高级`mem_effect`方言降级到`memory`方言，然后进一步降级到SMT-LIB方言，以利用最先进的内存语义SMT编码[20]。

**验证工具的构建：**
论文展示了如何利用这些语义方言构建三个与方言无关的验证工具：

1.  **翻译验证（Translation Validation）**：
    *   检查目标程序是否细化了源程序。当存在未定义行为时，非平凡细化很常见。
    *   该工具将源程序和目标程序都编译到语义方言，然后结合方言特定的细化关系，插入一个检查目标程序是否是源程序细化的断言。
    *   细化关系包括状态细化（内存和未定义行为标志）和函数结果细化。状态细化是通用的，而函数结果细化由用户为每种类型提供。
    *   实验中，该工具用于验证MLIR中的`arith-expand`、`arith-unsigned-when-equivalent`和`canonicalize`三个转换pass。通过对`arith`和`comb`方言定义语义，并进行穷举和随机测试，发现了`canonicalize` pass中的5个miscompilation bug，这些bug都已在MLIR上游中修复。这些bug通常是由于转换引入了不必要的“poison”值，导致代码行为变得未定义。

2.  **窥孔重写验证（Verifying Peephole Rewrites）**：
    *   窥孔优化器是编译器bug的常见来源。该工具用于形式化验证窥孔重写的正确性。
    *   **PDL到SMT的降级**：MLIR的PDL（Pattern Descriptor Language）用于声明式地定义重写规则。论文将PDL程序降级到SMT方言。
        *   `pdl.operand`和`pdl.attribute`被翻译为`smt.declare_const`。
        *   `pdl.operation`和`pdl.result`被翻译为相应操作的语义。
        *   `pdl.replace`被翻译为一个细化检查（通常是等式检查，忽略毒值）。
    *   **原生重写和约束**：PDL允许通过`pdl.apply_native_constraint`和`pdl.apply_native_rewrite`调用任意代码。论文假设用户为这些原生操作提供了对应的SMT编码，从而实现其在SMT层面的验证。
    *   **位宽枚举与独立位宽推理**：由于SMT不能推理参数化位宽的位向量，工具会迭代地为所有可行的位宽组合（例如最高64位）特化PDL模式并分别验证。对于某些情况，也实现了基于SMT整数的位宽独立推理，但这种方法在SMT中是不可判定的。
    *   实验中，论文重新实现了31个`arith`和35个`comb`方言的窥孔重写规则，并证明了它们的正确性。结果显示，大多数模式在64位内可以快速验证，而一些复杂模式需要更长时间。

3.  **数据流分析验证（Formal Verification of Dataflow Transfer Functions）**：
    *   数据流分析的传递函数（transfer functions）如果实现不正确，可能导致误编译。
    *   **`transfer` MLIR方言**：论文创建了一个`transfer` MLIR方言，用于定义数据流传递函数。该方言可以降级为C++代码以集成到编译器中，也可以降级为SMT方言进行机械推理。
    *   **伽罗瓦连接（Galois connection）**：通过伽罗瓦连接定义具体值和抽象值之间的关系，用于证明传递函数的健全性（soundness）和精度（precision）。
    *   例如，对于“已知位”（known bits）分析（判断位是0还是1），传递函数的健全性条件表述为：
        $$ \forall a, b \in A, \forall x, y \in C,  \text{wellFormed}(a) \land \text{wellFormed}(b) \land \text{includes}(a, x) \land \text{includes}(b, y) \Rightarrow \text{includes}(\text{abstractOp}(a, b), \text{concreteOp}(x, y))$$
        其中 `wellFormed` 和 `includes` 是为已知位域定义的谓词，`abstractOp` 是抽象操作，`concreteOp` 是具体操作。
    *   实验中，论文为CIRCT的`comb`方言实现了新的“已知位”和“需求位”（demanded bits）分析。新的“已知位”分析支持更多操作，并被证明在精度上显著优于CIRCT上游的分析（对RISC-V处理器设计，静态已知位增加了30%到55%）。新的“需求位”分析是CIRCT中未实现的全新分析。

**总结：**
这篇论文的核心在于，通过在MLIR框架内引入一套分层的、可扩展的语义方言，将形式化语义的定义提升为一等公民。这不仅使得编译器开发者能以熟悉的方式定义IR语义，还为SMT专家提供了优化语义编码的灵活性。最终，这一基础设施支持了与方言无关的、可重用的形式化验证工具的构建，并成功应用于发现真实编译器bug、验证优化规则和数据流分析的正确性，显著提升了MLIR生态系统中编译器工具的可靠性。
## Q&A

Q：怎样才能使得本文中的 translation validation 或者 peephole rewrite verification 做到更大的 scale？换句话说，scale up 所需的 manual effort 主要是什么？

A：本文中的 Translation Validation (翻译验证, TV) 和 Peephole Rewrite Verification (窥孔重写验证, PRV) 都依赖于将 MLIR 中的程序 dialect 转换为一组**语义 dialect** (semantic dialects)，然后将这些语义 dialect 进一步转换为 SMT-LIB 查询，最终由 SMT 求解器进行验证。因此，这两种验证方法在 scale up 时面临的核心挑战是**为更多 MLIR dialect 的操作定义形式化语义**，以及**处理日益增长的 SMT 查询复杂性**。

**Scale Up 所需的主要 Manual Effort (主要手动工作)**

根据论文的描述，scale up 所需的主要手动工作集中在以下几个方面：

1.  **定义和维护 Dialect 语义 (Defining and Maintaining Dialect Semantics):**
    *   **核心痛点：** 这是最主要的也是最繁重的工作。论文明确指出：“定义 IR 语义的核心策略是定义一套新的用于表达语义的 dialect——我们称之为语义 dialect。” (第3页) 并且，“赋予一个 dialect 语义就是定义一个从该 dialect 到我们语义 dialect 的**降低（编译转换）**。” (第8页)
    *   这意味着，对于 MLIR 中每一个需要验证的 `operation`，编译器开发者都需要**手动编写**一个 `lower` 过程，将其从原生的 MLIR operation 转换为语义 dialect 中的等效表达。这个过程通常是用 Python (如 xDSL) 或 C++ 实现的。
    *   **例子：** 论文中展示了 `arith.addi` 操作如何被转换为 `smt_bv.add` 以及 `poison` 相关的操作 (第9页)。对于每一个新的 `arith` 或 `comb` 操作，都需要进行这样的转换定义。
    *   **挑战：**
        *   **数量庞大：** MLIR 中有数百个甚至数千个 `operation` 分布在不同的 `dialect` 中。要覆盖 MLIR 生态系统中的所有重要 `dialect`，需要定义大量的语义转换。
        *   **语义复杂性：** 并非所有 `operation` 的语义都像 `arith.add` 那么简单。涉及到内存访问 (`memref` dialect)、控制流 (`scf` dialect)、并行计算 (`gpu` dialect) 等的 `operation`，其语义定义会非常复杂，可能需要引入更高级的语义 `dialect` (如论文中的 `mem_effect` 和 `ub_effect`)。
        *   **维护：** 随着 MLIR `dialect` 本身的演进，其 `operation` 的语义可能会发生变化，已经定义的语义也需要同步更新，这需要持续的人力投入。

2.  **处理 Native Constraints 和 Rewrites 的语义 (Handling Semantics of Native Constraints and Rewrites):**
    *   **背景：** 在 PRV 中，当使用 `pdl` dialect 定义重写规则时，`pdl.apply_native_constraint` 和 `pdl.apply_native_rewrite` 允许用户调用任意的宿主语言代码。
    *   **手动工作：** 论文提到：“为了将原生约束和重写转换为 `smt` dialect，我们**提供了到其相应 SMT 编码的映射**。” (第13页) 这意味着，这些任意的宿主语言代码逻辑，其形式化语义也必须**手动提供**给验证框架，而不是自动推断。
    *   **挑战：** 这引入了一个信任边界：验证工具信任这些手动提供的 SMT 语义是正确的。如果 `native` 代码的语义复杂，手动编写其 SMT 编码可能引入新的错误。

3.  **设计和实现高级语义 Dialect (Designing and Implementing Higher-Level Semantic Dialects):**
    *   **必要性：** 论文强调了多层语义 `dialect` 的重要性，以弥合程序 `dialect` 和低级 SMT-LIB 之间的语义鸿沟 (例如 `mem_effect`, `ub_effect`)。
    *   **手动工作：** 为新的复杂语言特性 (如并发、浮点数精度、自定义数据结构等) 设计和实现新的高级语义 `dialect` 及其操作，是一个高度专业化且需要大量手动思考和编码的工作，通常由形式化方法专家和 SMT 专家完成。这包括定义它们的 `denotational semantics` (如论文图 3-6 所示)，并实现它们到更低级 `dialect` (最终到 SMT-LIB) 的转换。

4.  **编写和验证数据流分析的 Transfer Functions (Writing and Verifying Dataflow Analysis Transfer Functions):**
    *   **背景：** 在数据流分析验证中，`transfer` dialect 用于定义 `transfer function`。
    *   **手动工作：** 论文中 `ORImpl` 的例子表明，这些 `transfer function` 是**手动编写**的 (`func.func @ORImpl(...)`)。虽然它们可以被验证，但 `transfer function` 本身的设计和实现（使其 sound 且 precise）是手动工作。
    *   **挑战：** `transfer function` 的正确性是数据流分析正确性的关键，编写它们需要深刻的领域知识和严谨性。
