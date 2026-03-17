---
aliases:
  - "Alive2: Bounded Translation Validation for LLVM"
tags:
  - PLDI21
  - translation-validation
  - LLVM
---

## 3 LLVM IR 语义的 SMT 编码

### 3.8 处理不支持的 LLVM Features

Alive2 不支持以下 LLVM 特性：
1. 异常，即 `invoke` 指令
2. 函数指针
3. volatile 变量
4. pointer-to-integer casts
5. 基于类型的别名分析
6. 大部分 LLVM 原语（一般起修饰作用，而非核心的 IR 指令）
> Out of the 258 non-experimental platform-independent intrinsics, we support 54 (21%).

> Beyond intrinsics, **LLVM has coarse-grained semantics for 463 library functions** including some from the C library, C++ library, Objective C runtime, etc. These specifications include **predicates** such as: the function always returns a non-null pointer, the function always/never returns, the function only reads/modifies objects referenced by the arguments, and so on. **Since LLVM optimization passes take advantage of these predicates to limit the scope of behavior of function calls, Alive2 has to mirror this knowledge.** For example, LLVM transforms a `printf()` of a constant string into a call to `puts()`; this looks like a failure of refinement unless we quip Alive2 with analogous knowledge about this pair of functions. So far **we have special-cased the semantics of 117 library functions, some of which only partially**.

对于 463 个库函数（from the C library, C++ library, Objective C runtime, etc.），LLVM 有粗粒度语义（以 spec 的形式提供）。对于其中的 117 个函数，Alive2 照搬了这些粗粒度语义。

> A major design goal of Alive2 is to avoid false alarms while attempting to verify as much as possible. Therefore, **when it encounters an unsupported feature, Alive2 attempts to produce an over-approximation of the semantics of that feature**.  For example, if we encounter a call to an unsupported intrinsic, we encode it as a call to an unknown function, which can potentially modify all the memory and return an arbitrary value. To avoid false alarms, we tag the function call as being an over-approximation. If later on Alive2 finds a bug in a transformation, we check whether an over-approximation was involved. To this end, **we record all SMT expressions produced when encoding over-approximated features**. Since  the SMT solver we use produces partial models (i.e., it may leave some variables unassigned when it declares a formula satisfiable if those variables can take any value without making the formula unsatisfiable), **we check if any expression from over-approximated features is in the model**. If not, it means the bug Alive2 found is real as it does not depend on the specifics of an over-approximated feature. Otherwise, if some expression is in the model, we cannot conclude anything: it may or may not be a bug in LLVM.

对于 Alive2 不支持的 LLVM feature，Alive2 会尝试对其进行上近似（如使用未解释函数）。但为了避免误报，Alive2 会将上近似所产生的所有 SMT 表达式打上标记。对于 solver 返回的反例 partial model，Alive2 在报告前先检查是否有被标记的 SMT 表达式在这个 model 当中。如果有，则说明当前反例可能是由上近似导致的，有误报的可能，因此丢弃该反例。

## 4. 内存的 SMT 编码

详见 [[An SMT Encoding of LLVM’s Memory Model for  Bounded Translation Validation]]。本文只考虑逻辑指针（即不考虑 integer-to-pointer 类型转换）以及单一地址空间。

**内存块**

内存分配的单元是内存块，每个内存块由 `bid` 唯一标识。

栈上变量和全局变量具有独立的块，`malloc` 则会创建一个新的内存块。

当循环展开后，程序中内存块的数量可以被静态确定（从而可以确定编码 `bid` 所需的位数）。

**指针**

指针表示为二元组 `(bid, off)` ，即内存块+块内偏移量，用 bit-vector 编码（拼接 `bid` 和 `off` ）。如果指针可以指向多个块（别名的情况），则需要每种别名情况分别创建 SMT 变量。`null` 用 `(0,0)` 表示。`undef` 指针用 `(β, ω)`表示，其中 β, ω 为 fresh 变量。

对于指针算术运算指令 `(gep ptr, i)`，返回一个指针，`bid` 不变，`off` 相加。如果算术用 `inbounds` 属性修饰，则还需要额外检查越界时返回 `poison`值的情况。 

**块属性与字节**

每个块都有附带的块属性，包含 size, alignment, is-read-only, is-alive, allocation-type, physical-address 等。

> To encode a memory block’s value, we use **an SMT array from pointer to byte**. **Bytes have three possible types: poison, pointer, and non-pointer.** A non-pointer byte uses an 8-bit bit-vector for the value, as well as an 8-bit mask to record which bits are poison. **Floats are converted to bit-vectors for storage.**

**内存访问**

> A possibly multi-byte load/store is split into single-byte loads/stores. These are then combined to produce a value of appropriate type (and size, as types’ sizes are not necessarily multiples of bytes).
> 
> The result of a load is poison if any of the following holds: (1) any of the loaded bytes is poison, (2) some of the loaded bytes have different types, (3) the load type does not match  the stored byte’s type (e.g., attempt to load a pointer from a block that has an integer stored).
> 
> We keep track of the undef variables used in stored values and pointers. These have to be replaced with fresh variables in each loaded value. Memory accesses using out-of-bounds  pointers or to already-freed blocks trigger UB. Store operations also trigger UB if the block is read-only.  
> 
> The `free(ptr)` operation updates the liveness of the corresponding block. It triggers UB if the block is already dead or not allocated on the heap, or if the offset is not zero.

主要是关于实现细节的描述，讨论 `undef`、 `poison`、UB 带来的复杂性。

## 6. 函数调用

**函数调用的基本语义**

一条 `call` 指令被视作一个（未解释的）纯函数：
- 它接收参数列表 (`args`) 和当前内存状态 (`M`) 作为输入
- 产生三个输出：一份可能被修改的新内存 (`M_o`)、返回值 (`v_o`)，以及一个表示调用是否触发 UB 的布尔标志位 (`ub_o`)。
- 最终表示为 `(M_o, v_o, ub_o) = call(fn, args, M)` 。

**SMT Encoding**

对于 source 中的每个调用，都**为其输出（内存、返回值、UB flag）分配独立的变量**。

（也就是说，在 SMT 建模中，它实际上并不是先定义一个表示调用的未解释函数 call，再令返回值为 call 在对应参数下的取值。而是绕开定义函数符号，直接为返回值创建独立的 SMT 变量。**这需要手动关联它们应该满足的函数的同余性**，详见下一段）

对于 target 中的每个调用，引入一个选择器变量 `i` ，其取值范围是 `0` 到 source 中调用 `f` 的次数 `|C_f_src|`：
- 如果 `i = j`(`0 <= j <= |C_f_src|-1` )，则表示这个 target 调用精化了 source 中的第 `j` 次调用
- 如果 `i = |C_f_src|`，则表示此 target 调用没有精化 source 中的任何调用。

用 ite 表达式编码目标值：target 调用的返回值被编码为一个大型的 ite 表达式，其分支由 `i` 的值决定：每个分支对应一个 source 调用的返回值。如果 `i` 指向了某个 source 调用，就采用那个调用的值；如果 `i = |C_f_src|`，则意味着这是一个“新”调用，与 source 中的调用无关，故不满足精化关系。

简单来说：**target 调用必须「认领」source 中的某一个调用作为其精化依据，并通过选择器变量在 SMT 公式中声明这种对应关系**。

**关联重复的调用**

LLVM 中存在一种常见的针对函数调用的优化：对于不进行内存读写的函数，**LLVM 会删除重复的函数调用**（例如，一个函数被标记为只读且无副作用，当它被相同的参数连续调用两次，且第二次调用在控制流上完全被第一次调用所支配，则可以安全地删除第二次调用）

这可能导致经过一个合法的优化 pass，target 与 source 中对相同函数的调用无法一一对应（例如，source 中的两个相同调用被优化 pass 去重，因此 target 中只有一个）。为了支持验证这种优化，Alive2 首先需要关联 source 中任意两个相同的函数调用，并为它们添加函数同余性约束：如果两次调用的输入等价，那么它们的输出也必须等价（理论上这里的等价可以替换为满足精化关系，但 LLVM 只会删除等价的重复调用，所以 Alive2 实现中也使用了更强的等价）。

![[Paper Reading/Alive2__assets/Figure 5.png|425]]

如果不做关联，由于两次的调用结果由两组不同的 SMT 变量表示，可能会导致伪反例。

**Limitations**

模型目前有一个简化假设：局部变量不会被函数调用修改。这是一种下近似，可能会漏掉一些涉及指针别名的复杂情况（如，局部变量的地址通过指针传递逃逸，从而被调用修改）。

## 7. 循环

Alive2 实现的是 bounded translation validation，对循环有限次展开，具体展开策略如下：
1. 首先执行 Tarjan-Havlak 循环分析算法，分析函数中循环的包含关系，并将循环表示为森林的结构（loop nesting forest）
2. 从内层向外层展开。这样可以保持展开后的指令数量与展开界限呈线性关系（避免从外到内展开，导致指令数量指数级增长）
3. 循环展开从实现上讲就是复制基本块。对于被复制的指令副本，需要相应填入 SSA 形式的操作数。
4. 对于 `jump` 跳转指令，还需要考虑跳转目标。设展开 `n` 次，则对于第 `1` 到 `n - 1` 个基本块，跳转目标为下一个基本块。对于第 `n` 个基本块，将其跳转目标设为一个特殊的 sink BB：sink BB 的可达条件被取反，合取到函数的前条件中，确保验证只考虑有限次迭代的路径
5. 除了复制循环所在的基本块，对于展开后的 CFG，还需要维护循环出口所在块的 $\phi$ 结点（用于显式合并分支结构的语句）。如果出口块的开头已经有 $\phi$ 结点，则只需要在已有的 $\phi$ 结点处填充新的基本块前驱；如果循环只有单一出口，且出口块支配（dominates）了循环块，那么只需要手动引入一个 $\phi$ 结点；对于需要引入多个 $\phi$ 结点的复杂情况，则采用回退策略，引入新的栈上变量表示。

要充分验证不包含循环的优化 pass，展开次数至少是 2 次。要充分验证循环优化，可能需要展开 64 次。