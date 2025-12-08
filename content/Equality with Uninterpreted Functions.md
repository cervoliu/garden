---
aliases:
  - EUF
tags:
  - logic
  - decision-procedure
---
将 Uninterpreted Functions 规约到 Equality Logic:
- Ackermann’s Reduction
- Bryant’s Reduction

将 UF 认为是 "abstract placeholder"，即不关心函数语义，对 UF 的要求仅有 function consistency ([[Congruence Relation]])。

EUF 的 decision procedure 是 [[Congruence Closure]] 算法，利用 equality graph 对公式化简与规约。
