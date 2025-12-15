---
aliases:
  - 等价饱和
tags:
  - equality-saturation
  - e-graph
  - compilation
  - optimization
---
最早的 publication: [POPL'09 Equality Saturation: a new approach to optimization](https://dl.acm.org/doi/10.1145/1480881.1480915)
利用 predefined rewrite rules 构建 [[e-graph]]，每个 e-graph 表示一个 congruence class

在 e-graph 上加入 rewrite rules 直到饱和 (e-graph 不再变化)

根据 cost model 对 saturated e-graph 做 optimal extraction  --> 得到经过 rewrite rules 等价变换后最优的结构。