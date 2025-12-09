---
tags:
  - concurrency
  - consistency-model
aliases:
  - 顺序一致性
  - 强内存模型
---
最早由 Lamport 在他 1979 年的论文中提出:

> the result of any execution is the same as if the operations of all the processors were executed in *some* sequential order, and the operations of each individual processor appear in this sequence in the order specified by its *program*.

中文译名为顺序一致性。SC 是最直观、最严格的内存模型，又被称为强内存模型。