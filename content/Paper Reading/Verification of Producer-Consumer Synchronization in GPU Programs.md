---
tags:
  - PLDI15
  - paper-reading
  - verification
  - GPU
zhihu-title: "[PLDI'15] Verification of Producer-Consumer Synchronization in GPU Programs"
zhihu-topics:
  - GPU
  - verification
  - producer-consumer
  - warp specialization
  - synchronization
zhihu-toc: true
feishushare: true
feishu_url: https://feishu.cn/docx/XWgydkzvqovgCixV78CcRg7An0d
feishu_shared_at: "2026-06-04 19:42"
zhihu-link: https://zhuanlan.zhihu.com/p/2045950375019259252
---
## 背景与动机

### Warp Specialization

传统的 GPU kernel 执行并行计算流水线的方式，是每个 CUDA core 各自独立地负责一部分数据，完整地跑完整条流水线。

而 warp-specialized kernel 则是不同的 warp 负责流水线的不同阶段，即根据流水线的阶段将 GPU 上的计算资源以 warp 为粒度进行 partition。例如，Part1 的 warps 专门负责将数据从 global mem 移动/拷贝到 smem，Part2 负责用 TMA loads 将 smem 中的数据搬运到 tensor core，Part3 负责调用 tensor core 算 GEMM，Part4 负责用 TMA stores 将计算结果写回 smem，Part5 负责 smem 写回 global mem，各个部分之间使用异步通信原语在 smem/global mem 上通信，遵循 Producer-Consumer 的异步并发模式。

值得注意的是，本文发表于 2015 年，彼时 GPU 尚不存在 TMA，tensor core 等新硬件，但 warp specialization 的概念早已有之（SC'11 cudaDMA）。

### 验证属性

> To perform synchronization between different warps, warp-specialized kernels use the producer-consumer **named barriers** available in PTX on NVIDIA GPUs.

本文仅考虑 named barrier 这一种同步通信原语，这在当时可能是除了整个 CTA 粒度的 `syncthreads()` 之外少数的几种同步原语(?)。named barrier 在当时的 NVGPU 上有 16 个，是真正的硬件资源，并且在一个 kernel 中可以重复利用。

对于 warp-specialized kernel，作者提出了三种值得验证的属性：
- **Deadlock Freedom**: named barriers 的使用不会导致死锁
- **Safe Barrier Recycling**：named barriers 的正确重用，即不可能出现上一轮同步未完成，下一轮就使用同名 barrier 的情况
- **Race Freedom**：named barriers 组织的并发对共享内存读写不出现数据竞争

事实上，作者后文定义了 Well-synchronized 属性，该属性同时蕴含 Deadlock Freedom 和 Safe Barrier Recycling，真正被检查的也是 Well-synchronized。

![[Paper Reading/Verification of Producer-Consumer Synchronization in GPU Programs__assets/Figure 2.png]]
### 验证的 kernel 边界

- 假设 1：kernel 中所有的控制流与数据流都是静态可分析的，且所有的 synchronization pattern 都是静态的
    - 控制流静态：每个线程可以表示成 straight-line code，没有动态的分支或循环
    - 数据流动态：对 global memory 与 shared memory 的访问可以视为以 blockIdx 和 threadIdx 为参数的函数
- 假设 2：所有的 CTA 执行相同代码，即 kernel 在 blockIdx 参数意义下完全对称。不考虑 inter-CTA 同步以及由 atomics 实现的 intra-CTA 同步。
	- 推论：只需要验证一个 CTA 就能得到整个 kernel 的验证结果。
- 假设 3：不考虑线程数据与 global memory 之间的读写，只考虑与 shared memory 的读写。
- 不假设 warp-synchronous execution：一个 warp 内的 32 个线程不一定锁步执行。

这些假设共同划定了论文试图解决问题的边界，它们的动机可以从两个角度理解：
- 从可行性角度，Ramalingam (2000) 的不可判定性结果表明，在任意控制流和同步存在的情况下不可能得到完备的验证方案。因此，若要同时获得 soundness 和 completeness ，必须对问题边界加以约束。
- 从工程实践角度，作者观察到这些假设在实际的 warp-specialized kernel 中几乎总是成立的。GPU 的 in-order 指令流水线使得动态分支代价极高，因此高性能 GPU kernel 本身就倾向于使用静态可分析的控制流。同样，文中已知的 warp-specialized kernel 确实只通过 shared memory(而非 global memory)进行线程间通信，因为 shared memory 的延迟远低于 global memory。排除 atomics 的理由也很实际：atomics 比 named barriers 慢至少一个数量级，且其验证被作者视为与 named barrier 验证正交的问题。

---

## 第三章：操作语义

### 3.1 语法——把 CUDA 程序抽象成「线程程序」

论文首先定义了一个核心语言，只保留与同步和共享内存访问相关的指令，其余全部抽象掉。一个线程程序的语法极其简单：

```
P ::= return | c; P
c ::= read g | write g | arrive b n | sync b n
```

只有四种语句：读/写共享内存变量 `g`，以及两种同步原语——非阻塞的 `arrive` 和阻塞的 `sync`。参数 `b` 是 named barrier 的编号（0–15），`n` 是期望到达该 barrier 的线程总数。`syncthreads` 在这个语法中就是 `sync 0 N`（所有线程在 0 号 barrier 上同步）。注意程序是 straight-line 的：命令之间用分号顺序连接，最后以 `return` 结束。

### 3.2 状态——两个映射刻画整个 CTA

一个 CTA 的运行状态由两个映射组成  (Sharma et al. 2015, 4)：

- **Enabled map $E$**：记录每个线程当前是否处于可执行状态（被 `sync` 阻塞的线程会被设为 `false`）。
- **Barrier map $B$**：对每个 barrier 记录三个信息：阻塞在该 barrier 上的线程列表 $I$、非阻塞到达的线程列表 $A$、以及期望的线程总数 $n$。未配置的 barrier 的 $n$ 记为 $\bot$。

初始状态是所有线程 enabled，所有 barrier 未配置。

### 3.3 小步语义——barrier 的完整生命周期

语义规则分两大类：CTA 级别的调度规则和单线程执行规则。最核心的是 barrier 的生命周期，分三个阶段。

**阶段一：配置。** 第一个到达某个 barrier 的线程负责"配置"该 barrier——将它的期望线程数设为 $n$。如果第一个到达者是 `arrive`，线程不阻塞，被加入 $A$ 列表；如果是 `sync`，线程被加入 $I$ 列表并被禁用。

**阶段二：收集。** 后续线程陆续到达同一个 barrier，各自被加入 $I$ 或 $A$ 列表。只要 $|I| + |A| < n$，barrier 就继续等待。

**阶段三：回收。** 当 $|I| + |A| = n$ 时，barrier 触发。所有阻塞在 $I$ 中的线程被重新启用，barrier 被重置为未配置状态 $([], [], \bot)$，可以立即被下一轮使用——这就是 **barrier recycling** 的关键机制。

此外还有两个错误规则：如果到达的线程数超过 $n$，或者某个线程声明的 $n$ 与已配置的 $n$ 不一致，则转入错误状态 `err`。

**关键设计选择**：这个语义不假设 warp-synchronous execution。论文明确指出这是刻意的——warp 内 lock-step 执行并非标准化行为，未来架构可能改变（事实上，自 Volta 架构引入 Independent Thread Scheduling 起，warp 内的 lock-step 假设正式被废除）。

---

## 第四章：验证算法

### 4.1 形式化定义——什么算"正确"

在描述算法之前，论文先给出了严格的定义体系。

**Trace（执行迹）**：从初始状态出发，依次应用语义规则得到的一系列配置。完整 trace 的终点要么是 `done`（所有线程正常退出），要么是 `err`（出错），要么是 deadlock（无规则可用且非 `done`）。

**Time 与 Happens-Before**：对于每条命令 $c_\eta$（对应程序点 $\eta$），定义 $t(\tau, \eta)$ 为该命令在 trace $\tau$ 中第几步被执行。Happens-before 关系 $R$ 定义为：$R(c_1, c_2)$ 当且仅当**在所有可能的 trace 中** $c_1$ 的执行时间都不晚于 $c_2$ 。

**Generation（代）**：对于同步命令 $c$，$Gen(\tau)(c)$ 表示在 trace $\tau$ 中 $c$ 所对应的 barrier 是第几次被回收后使用的。例如 generation 2 意味着该 barrier 此前已经被回收过一次。

**Well-Synchronized（良同步）**：一个 CTA 是 well-synchronized 的，如果**在所有可能的 trace 中**，每条同步命令的 generation 都相同且不为零。这个条件非常强——它同时蕴含了 deadlock-free（程序不会卡住）和 safe barrier recycling（barrier 的复用不会导致不同 generation 的线程混淆）。非零条件排除了 deadlock 和 error trace。

**Data Race Freedom**：在 well-synchronized 的前提下，如果两个访问同一共享变量的命令之间不存在 happens-before 关系，那么它们必须都是读操作，否则就是数据竞争。

### 4.2 WELLSYNC 算法——从一次具体执行推断所有执行

现在来到论文最精巧的部分。算法的核心思路可以概括为：**跑一次具体执行，得到一个 trace $\tau$，从中推导出 generation 映射和静态 happens-before 关系 $R$，然后检查这个 $R$ 是否足以保证所有 trace 中的 generation 与 $\tau$ 一致。**

用论文的示例来理解。Listing 2 是一个有两个 warp（64 线程）的 kernel，图 3 将其抽象为两个线程程序 $P$（thread 0）和 $Q$（thread 32）：

```
P:                          Q:
1  sync 0 64;     (1)      1  sync 0 64;     (1)
2  write g_0;              2  sync 1 64;     (1)
3  arrive 1 64;   (1)      3  read g_0;
4  sync 0 64;     (2)      4  sync 0 64;     (2)
5  sync 1 64;     (2)      5  write g_0;
6  read g_0                6  arrive 1 64    (2)
7  return                  7  return
```

算法先模拟一次执行得到 $\tau$。假设执行顺序是：$P$ 和 $Q$ 在 barrier 0 上同步 → $P$ 写 $g$、在 barrier 1 上 arrive、然后阻塞在 barrier 0 → $Q$ 在 barrier 1 上 sync（此时 barrier 1 完成回收）、读 $g$、在 barrier 0 上 sync（此时 barrier 0 完成回收）、写 $g$、在 barrier 1 上 arrive → $P$ 在 barrier 1 上 sync（barrier 1 第二次回收）、读 $g$。从这次执行中提取的 generation ID 已在上面用括号标出。

![[Paper Reading/Verification of Producer-Consumer Synchronization in GPU Programs__assets/Figure 5.png]]

接下来构造静态 happens-before 关系 $R$，分三步走：
- **第一步**（lines 4–6）：同线程内的顺序关系直接加入 $R$。图 5 中的实线边就是这些线程内边。
- **第二步**（lines 7–16）：跨线程的同步关系。如果 `arrive` 和 `sync` 属于同一 generation，加入 $(arrive, sync)$；如果两个 `sync` 属于同一 generation，加入双向边。图 5 中的虚线边对应这一步。
- **第三步**（line 17）：计算 $R$ 的传递闭包，得到完整的静态 happens-before 关系。

最后是验证环节（lines 18–23）：对每个 barrier 的相邻 generation，检查是否存在 happens-before 关系连接它们。具体来说，对于 generation $k$ 中 barrier $b$ 上的 `sync` 命令 $c_1$，以及 generation $k+1$ 中使用 barrier $b$ 的命令 $c_2$（其前驱命令为 $c_3$），检查 $(c_1, c_3) \in R$。这一步确保了 barrier 的每次回收与前一次使用之间有严格的 happens-before 间隔，从而 generation 在所有 trace 中保持一致。

对示例而言，我们需要确保 barrier 0 的 generation 1（$P_1$）与 barrier 0 的 generation 2（$Q_4$ 的前驱）之间存在路径，barrier 1 的 generation 1（$Q_2$）与 generation 2（$P_5$ 的前驱）之间也存在路径。从图 5 可以看出这些路径确实存在，因此该 CTA 是 well-synchronized 的。

### 完备性与数据竞争检测

定理 1 保证了 WELLSYNC 的 soundness：如果它返回 true，则 CTA 确实是 well-synchronized 的。Completeness 则来源于 well-synchronized 程序的 $R$ 必然是 precise 的（引理 1）。

一旦确认 well-synchronized 并获得了 $R$，数据竞争检测就变得简单：对于任意两个访问同一共享内存地址的命令，如果至少有一个是写操作，且 $(c_1, c_2) \notin R$ 且 $(c_2, c_1) \notin R$（即两者之间不存在 happens-before 关系），则报告数据竞争。在图 5 的示例中，所有共享内存访问之间都存在路径，因此是 race-free 的。

### WELLSYNC 验证算法的核心 insight

算法最巧妙的地方在于它避免了对所有可能交错进行穷举。它只跑**一次**具体执行，然后用静态分析验证这次执行中观察到的 generation 结构在所有交错下都保持不变。这之所以有效，是因为 well-synchronized 程序有一个关键性质：其**同步行为是确定性的**——无论线程如何交错调度，同一个 `sync` 命令在每次执行中总是和同一批线程同步，分配到同一个 generation。算法通过检查 happens-before 关系是否足以"隔离"相邻的 barrier generation 来验证这一性质，而这个检查本身的复杂度主要由传递闭包计算决定，在最坏情况下是 $O(n^3)$（$n$ 为命令总数）。

---

## 第五章：WEFT 验证工具

这一章解决的核心问题是：如何让一个理论上 $O(n^3)$ 的算法，在包含上千万条命令、需要执行上千亿次 race test 的真实 kernel 上实际跑得动。 答案分为两步：先做翻译（把 PTX 变成形式语言），再做优化（让算法可扩展）。
### 5.1 从 PTX 到线程程序——模拟执行一个 CTA

WEFT 的输入是 PTX 汇编代码，而非 CUDA 源码。这是经过深思熟虑的选择：PTX 是 CUDA 的中间表示，更接近硬件语义，named barrier 指令（`bar.sync` 和 `bar.arrive`）在 PTX 层面有精确的一一对应。

翻译过程本质上是一次**模拟执行**。WEFT 选择一个 CTA（通常是第一个），初始化基本寄存器（CTA ID、thread ID 等），但不假设 kernel 的任何输入参数。CTA 内的所有线程被并发模拟：每个线程独自向前推进，直到遇到 named barrier 操作则阻塞，等 barrier 完成后被唤醒继续执行。

在这一过程中，WEFT 自动完成三件事。第一，**展开所有静态绑定的循环**（statically bound loops）并**求值所有静态分支条件**，从而自动生成 straight-line 的线程程序——这正是第 3 章形式语言所需的形式。第二，跟踪通过 shared memory 和 Kepler 架构上 shuffle 指令传递的数据。第三，识别并记录下所有与同步和 shared memory 访问相关的指令，对应到形式语言中的 `sync`、`arrive`、`read`、`write` 命令。

模拟有三种可能的终止方式：
- **所有线程正常 `return`** → 成功生成线程程序，进入验证阶段。
- **检测到 deadlock** → 报告死锁位置和原因给用户。
- **WEFT 无法模拟某条必要指令** → 报告 kernel 超出验证范围。这通常发生在分支条件、shared memory 访问或 barrier 语句依赖于动态输入参数的情况下。但论文指出，在作者所知的所有 warp-specialized kernel 中，WEFT 都能成功生成 straight-line 线程程序。

此外，WEFT 还提供了一个可选的 **warp-synchronous 模拟模式**：同一 warp 内的线程以 lock-step 方式推进，任何线程到达 named barrier 即视为整个 warp 到达。这对应 NVIDIA PTX 手册中描述的实际硬件行为，但用户必须显式选择此模式，因为它并非未来的架构保证 。

### 5.2 优化后的验证算法——从 $O(n^3)$ 到可实用

直接实现第 4 章的算法（对全量命令计算传递闭包）在真实 kernel 上根本无法扩展。问题出在规模上：许多 kernel 包含数千万甚至上亿条命令，传递闭包是 $O(n^3)$ 的。WEFT 的优化策略基于一个关键洞察：**共享内存的读写操作不影响同步行为**——在验证 well-synchronized 性质时，可以安全地忽略所有 `read` 和 `write` 命令。

#### 第一步：Barrier Dependence Graph

WEFT 首先剔除所有内存访问指令，仅保留同步命令，然后模拟一次仅含同步命令的执行。在这个精简后的程序上运行 WELLSYNC 算法。由于命令数大幅缩减，传递闭包的计算变得可承受。

验证通过后，WEFT 构建 **barrier dependence graph**（barrier 依赖图）。每个已完成的 barrier（即一个特定的 generation）被转化为图中的一个节点，称为 **dynamic barrier**。以第 4 章的示例来说，trace $\tau$ 创建了四个节点：

- $n_1$：包含 $P_1$ 和 $Q_1$（barrier 0, generation 1）
- $n_2$：包含 $P_3$（arrive）和 $Q_2$（sync）（barrier 1, generation 1）
- $n_3$：包含 $P_4$ 和 $Q_4$（barrier 0, generation 2）
- $n_4$：包含 $P_5$（sync）和 $Q_6$（arrive）（barrier 1, generation 2）

然后沿着每个线程程序向前和向后遍历，建立 dynamic barrier 之间的 happens-before 和 happens-after 边。论文的图 6 和图 7 分别展示了图 2 和图 5 示例对应的 barrier dependence graph。关键性质是：**well-synchronized 的 CTA 的 barrier dependence graph 一定是 DAG**。

![[Figure 6&7.png]]
#### 第二步：以 Barrier 为锚点的区间分析

这是整个优化最核心的思想：不再为每条单独命令计算 happens-before 关系，而是**以 dynamic barrier 为锚点**，计算每个线程程序中各 barrier 节点之间的"区间"。

对于每个 dynamic barrier 节点，WEFT 计算它在每个线程程序中的 **latest happens-before point**（最晚的一定在此之前执行的点）和 **earliest happens-after point**（最早的一定在此之后执行的点）。图 8 直观地展示了这些关系：对于某个分析点，它在其他线程中的 happens-before 和 happens-after 锚点划定了一个"可与该点并发执行"的区间。

由于这一步只在 barrier dependence graph 上的节点之间计算传递闭包（而非在所有命令之间），时间复杂度降为 $O(D^2)$，其中 $D$ 是 dynamic barrier 的数量。而 $D$ 通常远小于总命令数（且同时活跃的 barrier 不超过物理上限 16 个），这使得计算量和存储量都大幅缩减。对于一个有 $N$ 个线程程序的 kernel，存储开销仅为 $O(DN)$，与总命令数无关。
#### 第三步：将 barrier 区间"下放"到每条命令

有了 barrier 级别的区间后，单条命令的 happens-before/happens-after 信息可以轻松推导出来：每条命令只需查看**同一线程中紧邻它的、不同物理 barrier 名的 dynamic barrier**，取它们的区间交集即可。例如，图 3 中 $P_2$（write）的 barrier 区间由它之前的 barrier 节点 $n_1$ 和之后的 barrier 节点 $n_2$ 共同确定。这一步耗时 $O(S)$（$S$ 为命令数），因为每个程序点只需查找常数个 barrier 节点。
#### 第四步：常数时间的 race test

最后，WEFT 枚举所有访问同一共享内存地址的命令对（至少含一个写操作），进行 race 检测。由于每条命令已经记录了它在所有其他线程程序中的 earliest happens-after 和 latest happens-before 点，判断是否存在数据竞争只需一次常数时间查表：如果另一条命令落在当前命令的 barrier 区间之外（即要么一定在此之前、要么一定在此之后），则无竞争；否则存在 race。

总结 WEFT 的 scalability insight：**将传递闭包从"命令 $\times$ 命令"的空间压缩到"barrier $\times$ 线程"的空间，而后续的所有操作或是线性的（对所有命令），或是常数时间的（每个 race test）。**

### 5.3 实验结果——在 26 个真实 kernel 上的表现

WEFT 被用于验证 13 个来自 CudaDMA 库的 kernel 和 13 个由 Singe 编译器生成的 kernel。CudaDMA kernel 的 CUDA 代码规模为 65–1561 行，而 Singe kernel 则达到 1684–13245 行。图 9 和图 10 汇总了全部结果。

**性能。** 大多数 kernel 在几分钟内完成验证（最长的是 hept visc fermi，约 7 分钟）。最极端的例子是 hept visc fermi，它大量使用 warp-synchronous 编程通过同一 shared memory 地址交换数千个常量，导致超过 1680 亿次 race test。WEFT 的内存消耗对多数 kernel 在数百 MB 量级，最大的 prf diff kepler 使用了约 5.1 GB——这在现代机器上完全可控。

**发现的 Bug。** 不出所料，所有 kernel 都是 well-synchronized 的（否则根本跑不起来）。但 WEFT 在 7 个 kernel 中发现了数据竞争，分为两类：

**良性竞争（Benign Races）。** 所有 RTM kernel 的初始化循环中存在有意设计的多线程写同一值的模式——多个线程从 global memory 加载相同数据并写入 shared memory 的同一位置。这在语义上无害，且使代码更简洁。WEFT 不区分良性与恶性，正确地将它们全部报告出来，因为即使是良性竞争也可能造成性能退化。

**有害竞争（Harmful Races）。** 两类 kernel 中发现了真正的 write-after-read bug：
- **双阶段 RTM kernel**（CudaDMA）：Listing 3 揭示了问题——compute warp 的主循环中，`start_async_dma` 调用（触发 DMA warp 向 shared memory 写入）被放置在 compute warp 最后一次读取同一 shared memory 缓冲区之前。结果 DMA warp 可能在旧数据被消费完之前就开始覆盖它。这个 bug 对应一个困扰了两年的非确定性 bug，但一直未修复，因为单阶段 kernel 性能更好、用得更多 。

- **DME 和 Heptane chemistry kernel**（Singe，Kepler 架构）：Singe 编译器的 shared memory 空间管理使用了一个启发式算法求解 NP 完全的 bin-packing 问题，在 Kepler 架构上由于更大的寄存器文件空间允许了更激进的映射策略，触发了该启发式算法的 bug，导致 write 可能在 read 之前发生。这个 bug 在现有硬件上由于中间指令足够多而从未实际触发，但无法保证未来架构不会暴露它。