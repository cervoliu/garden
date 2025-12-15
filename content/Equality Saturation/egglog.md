---
tags:
  - equality-saturation
  - term-rewrite
  - database
---

推荐阅读: [egglog tutorial (EGRAPHS 2023) on YouTube](https://www.youtube.com/watch?v=N2RDQGRBrSY)。

> egglog = [[egg (e-graphs good)|egg]] + datalog, but it is strictly superior to both egg and datalog.

e-matching 中，一个 rewrite rule 可能有多个 pattern(?x, ?y, ...)，在 egg 中，multi-pattern e-matching 效率比较低下。egglog 借助了 database 中的 generic join 来加速这一过程。