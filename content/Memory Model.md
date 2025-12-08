---
aliases:
  - Memory Consistency Model
  - 内存模型
tags:
  - memory-model
  - consistency-model
---
内存模型是对程序运行环境中内存[[Operational Semantics|操作语义]]的形式化描述。根据描述的运行环境层次，内存模型可分为硬件与软件内存模型两类。

硬件内存模型刻画了处理器架构层面的内存语义，如 [[Total Store Order]]、[[Partial Store Order]]、[[POWER memory model|POWER]]、[[ARM memory model|ARM]]。

软件内存模型则定义了程序语言、操作系统层面的内存一致性规范，如 Java 内存模型、C11 内存模型和 [[Linux Kernel Memory Model|Linux 内核内存模型]]。
