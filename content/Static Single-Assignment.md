---
aliases:
  - SSA
  - 静态单赋值
tags:
  - compilation
---
一种 IR 设计哲学，中文称为 静态单赋值，要求有两条：

1. 每个变量恰好被赋值一次 (single)
2. 赋值唯一性在编译期（“静态”）即被确定 (static)

与之相对的概念有 运行时（“动态”）单赋值 (dynamic single-assignment)。