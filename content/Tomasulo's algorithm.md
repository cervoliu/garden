---
tags:
  - out-of-order
  - architecture
  - algorithm
---
一种为乱序执行 CPU 设计的动态指令调度算法

核心 idea 包括 register renaming，执行单元各自的保留站（reservation stations），用于保持 program instruction order 的 reorder buffer，以及用于广播数据到保留站（data forwarding）的公共数据总线（common data bus, CDB）


学习资料：
- [Out-of-Order Execution(Tomasulo's Algorithm)](https://www.youtube.com/watch?v=EzEKGlO9w4Y)
- [浅谈乱序执行 CPU（一：乱序）](https://jia.je/hardware/2021/09/14/brief-into-ooo)