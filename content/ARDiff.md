---
title: "ARDiff: Scaling Program Equivalence Checking via Iterative Abstraction and Refinement of Common Code"
tags:
  - FSE20
  - paper-reading
  - equivalence-checking
  - CEGAR
feishushare: true
feishu_url: "https://feishu.cn/docx/Mhexdulqmo69QoxtJjLcnc10n9f"
feishu_shared_at: "2026-03-03 16:13"
---
> In the context of functional equivalence checking, two main approaches for dealing with these limitations of symbolic execution have been proposed: **Differential Symbolic Execution (DSE)** [38] **uses uninterpreted functions** - function symbols that abstract internal code representation and only guarantee returning the same value given the same input parameters - **to abstract syntactically identical segments of the code** in the compared versions.

Differential Symbolic Execution (DSE): 用 UF 抽象掉完全相同的代码片段

> **IMPacted Summaries (IMP-S).** Instead of identifying common code blocks, Bakes et al. [6] propose a technique that uses static analysis, namely, forward and backward control- and data-flow analysis, to identify all statements impacted by the changed code. The tool then prunes all clauses of symbolic summaries that do not contain any impacted statements.

Impacted Summaries (IMP-S): 基于静态依赖分析的 path summary 剪枝

ARDiff 一句话省流版：DSE + CEGAR


![[Architecture.png]]

但没有从本质上解决从 DSE 处继承的缺陷：对于无法被抽象掉的循环，仍然依靠循环展开。论文中对方法 validity 的讨论只讨论了 CEGAR（上近似部分），对循环展开避而不谈，个人认为是有严重问题的。