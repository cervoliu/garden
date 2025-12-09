---
aliases:
  - EUF
tags:
  - logic
  - decision-procedure
---
EUF Theory 是一阶逻辑中的一种基础[[First-Order Theory|理论]]，定义如下：
- 签名：等号谓词 $=$，任意常量符号和任意函数符号
- 公理：等号的自反、对称、传递性；函数的一致性。

将函数认为是「未解释的」，即不关心函数语义，对函数的要求仅有一致性 (即 $=$ 构成[[Congruence Relation|同余关系]])。

可以将 Uninterpreted Functions Theory 规约到 Equality Theory:
- Ackermann’s Reduction
- Bryant’s Reduction

EUF 的 decision procedure 是 [[Congruence Closure]] 算法，利用 equality graph 对公式化简与规约。
