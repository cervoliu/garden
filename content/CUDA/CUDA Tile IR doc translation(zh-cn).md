---
draft: true
---
# 0. 译者注

- Operations 在手册中一般指 Tile IR 指令，故翻译为“指令”。
- tile, block, grid, kernel 等领域特定概念不做翻译。

# 2. 编程模型

Tile IR 扩展了 CUDA 的底层编程模型，引入了不同于 CUDA C++ 或 PTX 中已有内容的新抽象。
本节介绍 Tile IR 的编程模型，并让读者熟悉其核心概念和抽象。我们将通过一系列实际程序来实现这一点，最终构建一个动态、高性能的 GEMM 实现，该实现利用了 Tile IR 的所有主要特性。

Tile IR 拥有一个具备一定表达能力的基于 tile 的编程模型。我们将通过逐步调整示例程序，使其更好地利用 Tile IR 的特性（这些特性有助于简化程序并让编译器提供性能可移植性），来向用户介绍表示张量计算的各种方法。我们首先介绍读者可能从现有技术中已经熟悉的概念。

## 2.1. Tile 内核

Tile IR 程序被称为 *tile kernel*，与 CUDA C++ 或 PTX 类似，它们是函数，在调用时作为 *N* 个副本并行运行。主要区别在于 Tile kernel 执行的基本单位是 *tile block*，它表达了单个逻辑 tile 线程在数据的多维 tile 上执行的计算。

在执行期间，每个 tile kernel 被称为一个 tile kernel 实例。

下面是一个简单的 Tile IR kernel，它打印 “Hello World!”。

```python
cuda_tile.module @hello_world_module {
	entry @hello_world_kernel() {
	print "Hello World!\n"
	}
}
```

对于那些熟悉 CUDA 线程的读者，需要注意 Tile IR 的线程是不同的概念。在我们进一步深入形式化描述之前，必须强调它们的差异。

Tile kernel 是程序的入口点，作为 tile block 的并行实例执行。

### 2.1.1. tile programming 有何不同？

Tile IR 是 CUDA 编程模型的扩展，它实现了对 tile 编程的 first-class 支持。tiled kernels 将程序表示为在 tile 上操作的逻辑 tile 线程的 grid。Grid 和单个 tile 线程到底层硬件线程的映射被抽象出编程模型之外，由编译器处理。

NVIDIA 流式多处理器（SM）的 SIMT 编程模型是一种线程在（相对）较小的数据块上操作的模型，用户负责将线程划分和调度到适当的块中，以高效地计算输入数据。该模型为程序员提供了将线程映射到数据（或反之）的灵活性。SIMT 是 CUDA 和 PTX 公开的编程模型，自 2006 年推出以来，一直为 NVIDIA GPU 提供良好服务。

深度学习重要性的上升既为用户工作负载带来了更大的规律性，也带来了为这些工作负载提供性能的日益增长的需求。正如[引言](introduction.html#section-introduction)中所讨论的，这导致了以 tensor cores 形式出现的新专用硬件。

Tensor Core 为 SIMT 编程模型引入了一个新维度。现在，SM 线程必须与 tensor cores 协作才能达到峰值性能。随着每一代新硬件的出现，这两块硅片之间的相互作用释放了惊人的新性能，但也增加了编程复杂性。

Tile IR 的构建旨在帮助实现充分利用底层硬件的能力的高性能算法，同时尽量减轻编程复杂性。

与传统的 SIMT 模型相比，Tile IR 通过抽象线程到数据的映射简化了如 tensor core 的专用硬件的使用。
### 2.1.2. Kernel I/O

为了说明 Tile IR 的设计，我们将从简单的 hello world 内核转向一个实现一维张量（即向量）加法且具有固定块大小 `128` 的内核。本节中展示的所有示例都可以在[编程模型示例程序](appendix.html#section-appendix-sub-prog)中找到。

tile kernel 接受输入和输出作为参数；这是消费和产生数据的唯一机制，因此我们首先定义 kernel 参数。

```python
entry @vector_block_add_128x1_kernel(
%a_ptr_base_scalar : !cuda_tile.tile<ptr<f32>>,
%b_ptr_base_scalar : !cuda_tile.tile<ptr<f32>>,
%c_ptr_base_scalar : !cuda_tile.tile<ptr<f32>>)
```

上面的代码片段定义了一个名为 `vector_block_add_128x1_kernel` 的 kernel，它接受三个参数，代表两个输入缓冲区 `a` 和 `b`，以及输出缓冲区 `c`。所有参数的类型都是标量指针，表示为包含单个指针的零维张量。

Tile IR 中的所有值要么是张量，要么是张量视图（参见[张量视图](types.html#type-views)）。张量是一个 n 维矩形数组，由其秩（维度数量）、形状（每个维度上的范围）及其基本元素类型描述。张量的秩可以是 0 或更高。秩为 0 的张量是标量。秩、维度和元素类型都是张量类型的一部分，并且是静态已知的。张量类型被分配给代表全局内存中包含的多维数组的逻辑视图的值。全局内存总是通过张量访问，因此指针参数总是指向全局设备内存中的 CUDA 设备分配。tile kernel 没有返回值，因此省略了返回类型注释（请注意，tile 函数可能有返回类型，将在后面讨论）。

细心的读者现在可能想知道，为什么我们给输入和输出标量指针类型，而不是具有静态已知秩、形状和步长的张量类型。Tile IR 编程模型中的一个常见模式是内核接受非结构化的基指针作为参数，然后可以用来构建所需的张量。这种灵活性产生了多种使用指针的方式，具体取决于所需的程序行为，但我们将首先关注最灵活的表示方式，将基指针转换为任意指针的张量。任意指针的张量是一种灵活的抽象，允许您一次性从一组地址执行分散/聚集式加载，并将一组地址视为逻辑 tile。

我们必须采取几个步骤将基指针 `%a_ptr_base_scalar` 转换为代表我们要计算的 `128x1` tile的指针张量，该tile包含来自 `(base + 0, ..., base + 127)` 的地址。

我们首先创建一个偏移张量，表示包含区间 `(0, 127)`。我们使用 [cuda_tile.iota](operations.html#op-cuda-tile-iota)，它构造一个范围张量，从 `0` 计数到 `n - 1`，形成一个大小为 `n` 的向量。

```
%offset = iota : tile<128xi32>
```

  

然后我们将标量 `ptr<f32>` 重塑为一维张量 `1xptr<f32>`，以便获得正确的秩。

  

```

%a_ptr_base_tensor = reshape %a_ptr_base_scalar :

tile<ptr<f32>> -> tile<1xptr<f32>>

```

  

然后我们广播指针，得到一个包含 128 个元素的一维张量 `(base, ..., base)`。

  

```

%a_ptr = broadcast %a_ptr_base_tensor : tile<1xptr<f32>> -> tile<128xptr<f32>>

```

  

将偏移张量添加到指针张量，得到一个 `tile<128xptr<f32>>`，其值包含 `(base + 0, ..., base + 127)` 的指针。现在我们有了一个代表我们要计算的tile的指针张量。我们对 `a`、`b` 和 `c` 执行相同的步骤。

  

```

%a_tensor = offset %a_ptr, %offset :

tile<128xptr<f32>>, tile<128xi32> -> tile<128xptr<f32>>

```  

最后，我们加载两个操作数，执行加法，并存储到输出。

```

%a_val, %token_a = load_ptr_tko weak %a_tensor : tile<128xptr<f32>> -> tile<128xf32>, token

%b_val, %token_b = load_ptr_tko weak %b_tensor : tile<128xptr<f32>> -> tile<128xf32>, token

%c_val = addf %a_val, %b_val rounding<nearest_even> : tile<128xf32>

store_ptr_tko weak %c_tensor, %c_val : tile<128xptr<f32>>, tile<128xf32> -> token

```

  

现在我们有了一个完整的单tile块内核，它在 128 元素向量上执行加法。如您所见，这段代码是从单个控制线程编写的，但其并行级别将由编译器决定。

  

内核通过指针参数与全局内存通信，这些参数可以通过形状和步长操作转换为张量，以实现灵活的数据访问。

# 6. 语义

本节对 Tile IR 的操作语义进行书面阐述。这些语义旨在帮助以下两类读者理解 Tile IR：1) 对将 Tile IR 作为代码生成目标感兴趣的人员；2) 对阅读他人编写的 Tile IR 程序感兴趣的人员。

本节并不试图形式化所有可能的行为，无论这些行为是否符合公理化公式或小步操作语义。如需更深入地了解该语言及其核心概念，请参阅“编程模型”部分。

我们首先介绍抽象机器状态和语言定义，然后描述各个内核和程序的语义。接下来，我们讨论更多操作的语义，有关所有操作及其行为的完整描述，请参阅“Operations”部分。

## 6.1 抽象状态机

Tile IR 抽象状态机的状态 $\mathcal{S}$ 是一个元组，由以下几个部分组成：
- 一个 well-formed 的模块（Module） $\mathcal{M}od$ ，其中模块的概念见 [[#6.2 模块]]。
- 一个 tile block（或逻辑 tile thread）的 grid $\mathcal{TB}$，每个 grid 都表示一个 tile-kernel 实例。
- 每个 tile block 都有一个无限寄存器文件（infinite register file）$\mathcal{R}$，将寄存器的名称映射到对应的值。
- 一个全局内存 $M$，将地址映射到标量值。
- 一组待定的内存访问集合 $P$，$P$ 的进展与 Tile IR 操作的执行异步进行。
## 6.2 模块

# 7. 内存模型

内存模型定义了 loads 从内存中可以取出的合法的值。这并不像乍一看那样简单，为了实现编译器和硬件优化，我们允许指令重排序。

Tile IR 内存模型派生于 PTX 内存模型，因此它们的同步原语（synchronization primitives）有意保持相似。

## 7.1 内存模型操作

内存模型由 tile operations 中「独立元素访问之间的关系」以及「对这些关系构成的环的限制」组成。因此，一条 Tile IR 内存指令会生成一个或多个内存模型操作。具体地，tile loads, stores。
## 7.2 作用域

| 作用域          | 描述                                  |
| ------------ | ----------------------------------- |
| `tile_block` | Tile block 作用域，用于单个 tile block 内部通信 |
| `device`     | Device 作用域，用于单个 GPU 内部通信            |
| `sys`        | System 作用域，用于系统间通信                  |
## 7.3 内存序

## 7.4 Moral Strength

