---
aliases:
  - 内存模型
  - 内存连贯性模型
  - 内存一致性模型
  - Memory Model
tags:
  - memory-model
  - consistency-model
  - concurrency
  - architecture
---
推荐阅读: 
- [内存模型与内存序 by jiege](https://jia.je/hardware/2024/09/04/memory-model-and-memory-ordering)
- [Hardware Memory Model by Russ Cox](https://research.swtch.com/hwmm)
- [Programming Language Memory Models by Russ Cox](https://research.swtch.com/plmm)。

一些对内存模型是什么的回答：
- 内存模型是对程序运行环境中内存[[Operational Semantics|操作语义]]的形式化描述
- 内存模型定义了并发系统中内存操作的可见性规则与顺序约束
- 内存模型规定了不同线程访问共享内存时的交互规范，从而确定了程序的合法执行集合
- 对共享内存系统（共享内存，还可能包含 write buffer，cache 等其他存储）的行为进行定义，将 Load/Store 指令流作为该系统的输入，定义 Load 指令允许获取的值即限制了不同内存模型的行为
	- 举例来说，如果在所有的观测下系统都满足 SC 模型，即使在硬件层面发生了乱序，也认为系统的内存模型就是 SC
	- 内存模型是一种基于观测定义的概念模型，而并非物理模型

[[Sequential Consistency|顺序一致性模型 SC]] 是最严格的内存模型，因此也被称为「强内存模型」。然而，SC 在实际运行场景下的严格实现会显著限制编译器、硬件的优化空间。因此，现代并发系统普遍采用缓存、流水线等技术放松内存操作的顺序约束，从而允许内存操作的乱序执行以提升性能——刻画这样的并发系统的内存模型称为「弱内存模型」。

根据描述的并发系统层次，弱内存模型可分为硬件内存模型与软件内存模型：
- 硬件内存模型反映处理器架构的内存访问行为，通过刻画处理器架构中采用的流水线、写缓存、推测执行等硬件技术，允许了内存操作的乱序执行。主流硬件内存模型包括 [[Total Store Order|TSO]]、[[Partial Store Order|PSO]]、[[POWER memory model|POWER]]、[[ARM memory model|ARM]] 等。
- 软件内存模型反映了由编程语言或操作系统定义的内存语义，提供了内存屏障、原子操作等同步原语供程序开发者显式控制内存操作的顺序。主流软件内存模型包括 Java 内存模型、[[C11 Memory Model|C11 内存模型]]、 [[Linux Kernel Memory Model|Linux 内核内存模型]]等。
	