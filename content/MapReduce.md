---
title: "MapReduce: Simplified Data Processing on Large Clusters"
tags:
  - distributed-system
  - paper-reading
  - OSDI04
---
MapReduce 是一种用于处理和生成大规模数据集的编程模型及其相关实现。它抽象了并行化、容错、数据分发和负载均衡等复杂细节，使得不具备分布式系统经验的程序员也能利用大型集群资源。

**1. 编程模型 (Programming Model)**
用户通过定义两个函数来表达计算：
*   **Map函数 (Map function)**：用户编写，接收一个输入 `key/value` 对 $(k_1, v_1)$，并产生一组中间 `key/value` 对 $(k_2, v_2)$。类型签名为：$\text{map}(k_1, v_1) \rightarrow \text{list}(k_2, v_2)$。
*   **Reduce函数 (Reduce function)**：用户编写，接收一个中间 `key` $k_2$ 和与该 `key` 关联的一组 `value` 列表 $\text{list}(v_2)$。它将这些 `value` 合并成一个（通常更小）的 `value` 集合。类型签名为：$\text{reduce}(k_2, \text{list}(v_2)) \rightarrow \text{list}(v_2)$。
MapReduce 库将所有具有相同中间 `key` 的 `value` 集合提供给 `Reduce` 函数。典型的例子是字数统计：`Map` 函数输出 $\langle \text{word}, 1 \rangle$ ，`Reduce` 函数则对同一单词的所有计数求和。

**2. 实现概述 (Implementation Overview)**
该实现针对 Google 的集群环境设计，特点包括：商品化 PC (双处理器 x86, 2-4GB 内存)，100Mbps/1Gbps 以太网，数千台机器组成集群，机器故障常见，存储由直接连接的廉价 IDE 磁盘通过分布式文件系统 GFS (Google File System) 提供可靠性。

**2.1 执行流程 (Execution Overview)**
1.  用户程序中的 MapReduce 库将输入文件分割成 `M` 个分片 (split) (通常 16MB-64MB)，并在集群中启动程序的多个副本。
2.  其中一个副本被指定为 `Master`，其余为 `Worker`。`Master` 负责调度 `M` 个 `Map` 任务和 `R` 个 `Reduce` 任务。
3.  被分配 `Map` 任务的 `Worker` 读取对应的输入分片，解析 `key/value` 对并传递给用户定义的 `Map` 函数。`Map` 函数产生的中间 `key/value` 对被缓冲在内存中。
4.  缓冲的中间对定期被写入 `Map Worker` 的本地磁盘，并根据分区函数 `hash(key) mod R` 被划分为 `R` 个区域。`Map Worker` 将这些区域的位置和大小报告给 `Master`。
5.  当 `Reduce Worker` 收到 `Master` 发来的中间数据位置信息后，通过 RPC 从 `Map Worker` 的本地磁盘读取这些数据。读取完成后，`Reduce Worker` 根据中间 `key` 对数据进行排序，确保相同 `key` 的 `value` 集中在一起。如果数据量过大，则使用外部排序 (external sort)。
6.  `Reduce Worker` 遍历排序后的中间数据，对于每个唯一的中间 `key`，将其和对应的 `value` 列表传递给用户定义的 `Reduce` 函数。`Reduce` 函数的输出追加到最终的输出文件。
7.  所有 `Map` 和 `Reduce` 任务完成后，`Master` 唤醒用户程序，MapReduce 调用返回。
最终输出存储在 `R` 个文件中，每个 `Reduce` 任务一个。

**2.2 Master 数据结构 (Master Data Structures)**
`Master` 维护每个 `Map` 任务和 `Reduce` 任务的状态（空闲、进行中、完成）及其所在 `Worker` 机器的身份。对于每个完成的 `Map` 任务，`Master` 存储其生成的 `R` 个中间文件区域的位置和大小，并将其增量推送给进行中的 `Reduce` 任务。

**2.3 容错 (Fault Tolerance)**
*   **Worker 故障 (Worker Failure)**：`Master` 定期 `ping` `Worker`。如果 `Worker` 无响应，`Master` 将其标记为失败。该 `Worker` 完成的 `Map` 任务会被重置回空闲状态并重新调度，因为其输出存储在本地磁盘上，在 `Worker` 失败后变得不可访问。进行中的 `Map` 或 `Reduce` 任务也会被重置和重新调度。已完成的 `Reduce` 任务不需要重执行，因为其输出存储在全局文件系统 (GFS) 中。
*   **Master 故障 (Master Failure)**：`Master` 可以周期性地对自身数据结构进行检查点 (checkpoint)，从最近的检查点恢复。但由于 `Master` 只有一个，其失败概率较低，当前实现会在 `Master` 失败时中止 MapReduce 计算。
*   **失败时的语义 (Semantics in the Presence of Failures)**：
    *   如果用户提供的 `Map` 和 `Reduce` 操作是其输入的确定性函数 (deterministic functions)，分布式实现会产生与无故障顺序执行相同的输出。这通过原子提交 (`atomic commits`) 实现：任务将输出写入私有临时文件，完成后原子性地重命名。
    *   如果操作是非确定性的 (non-deterministic)，MapReduce 提供较弱但合理的语义：某个 `Reduce` 任务 $R_1$ 的输出等同于非确定性程序顺序执行 $R_1$ 产生的输出。但不同的 `Reduce` 任务 $R_2$ 的输出可能对应于该非确定性程序的不同顺序执行。

**2.4 本地性 (Locality)**
为节省网络带宽，`Master` 调度 `Map` 任务时会优先选择包含相应输入数据副本的机器。如果无法实现，则选择与数据副本位于同一网络交换机下的机器。这使得大部分输入数据能在本地读取，减少网络负载。

**2.5 任务粒度 (Task Granularity)**
`Map` 阶段被划分为 `M` 个分片，`Reduce` 阶段划分为 `R` 个分片。`M` 和 `R` 通常远大于 `Worker` 机器数量，以提高动态负载均衡并加速故障恢复。实际中 `M` 通常根据输入数据量（例如每个任务 16MB-64MB）确定，`R` 则为预期 `Worker` 机器数量的几倍。

**2.6 备份任务 (Backup Tasks)**
为缓解“拖沓者” (straggler) 问题（即少数机器因各种原因导致任务完成时间异常长），当 MapReduce 操作接近完成时，`Master` 会调度剩余进行中任务的备份执行 (backup executions)。任务一旦主要执行或备份执行完成，即被标记为完成。这会略微增加计算资源消耗，但显著减少总完成时间。

**3. 优化与特性 (Refinements)**
*   **分区函数 (Partitioning Function)**：用户可指定自定义分区函数来控制中间 `key` 如何分发到 `R` 个 `Reduce` 任务，例如，将同一主机的所有 URL 放入同一个输出文件。
*   **排序保证 (Ordering Guarantees)**：MapReduce 保证在一个给定分区内，中间 `key/value` 对是按 `key` 递增顺序处理的。这便于生成有序输出文件。
*   **Combiner 函数 (Combiner Function)**：可选的 `Combiner` 函数在 `Map Worker` 机器本地执行，对 `Map` 任务产生的中间数据进行部分合并，以减少发送到 `Reduce` 任务的网络数据量。`Combiner` 通常与 `Reduce` 函数代码相同，但其输出是发送到 `Reduce` 任务的中间文件。
*   **输入/输出类型 (Input and Output Types)**：MapReduce 库支持多种输入格式（如文本模式、按 `key` 排序的 `key/value` 对序列），并允许用户添加自定义格式。输出也支持多种格式。
*   **副作用 (Side-effects)**：用户代码可以产生辅助文件作为额外输出，MapReduce 依赖应用程序作者确保这些副作用的原子性和幂等性。
*   **跳过不良记录 (Skipping Bad Records)**：MapReduce 提供一种可选模式，检测并跳过导致 `Map` 或 `Reduce` 函数崩溃的特定记录，以使计算继续进行。
*   **本地执行 (Local Execution)**：提供一种替代实现，可在本地机器上顺序执行 MapReduce 操作，便于调试和测试。
*   **状态信息 (Status Information)**：`Master` 运行内部 HTTP 服务器，提供状态页面展示计算进度、资源使用情况、任务日志链接以及故障信息。
*   **计数器 (Counters)**：MapReduce 库提供计数器机制，允许用户代码统计各种事件（如处理的单词总数）。计数器值从 `Worker` 定期传回 `Master` 并进行聚合，排除重复执行的影响。

**4. 性能 (Performance)**
*   **Grep**：扫描 1TB 数据查找特定模式，150秒完成。峰值输入读取速率超过 30GB/s。
*   **Sort**：排序 1TB 数据，891秒完成。输入读取峰值约 13GB/s，数据 Shuffle 峰值约 10GB/s，输出写入峰值约 2-4GB/s。本地性优化使得大部分数据在本地读取，节省网络带宽。
*   **备份任务影响**：禁用备份任务，排序时间增加 44%（1283秒）。
*   **机器故障影响**：在排序过程中故意杀死 200 个 `Worker` 进程，总完成时间仅增加 5%（933秒），体现了强大的容错能力。

**5. 经验 (Experience)**
MapReduce 已在 Google 内部广泛应用，涵盖大规模机器学习、聚类、数据提取、图计算等。截至 2004 年 9 月，已有近 900 个独立的 MapReduce 程序。它使得不熟悉分布式系统的程序员也能高效利用大量计算资源，显著加快了开发和原型周期。Google 的生产索引系统已完全重写为 MapReduce 流程，代码量减少，易于变更和操作。

**6. 结论 (Conclusions)**
MapReduce 模型的成功归因于：
1.  **易用性**：隐藏了并行化、容错、本地性优化和负载均衡的复杂性。
2.  **广泛适用性**：大量问题可轻松表达为 MapReduce 计算。
3.  **高性能**：实现可扩展到数千台机器的集群，高效利用资源。
通过这项工作，作者们学到：限制编程模型有助于并行化和分布式计算并实现容错；网络带宽是稀缺资源，系统优化应减少网络数据传输；冗余执行可降低慢速机器的影响并处理机器故障和数据丢失。