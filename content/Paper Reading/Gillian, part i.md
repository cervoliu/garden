---
title: "Gillian, part i: a multi-language platform for symbolic execution"
tags:
  - PLDI20
---
原文链接: [Gillian, part i: a multi-language platform for symbolic execution](https://dl.acm.org/doi/pdf/10.1145/3385412.3386014)

Gillian 符号执行框架最早的 pub，核心的 idea 我觉得应该就是 parametric symbolic execution。这里的 parametric 是 with respect with memory model 的。不过并不是弱内存一致性那个 memory model，就是一般意义的程序如何与内存交互的建模。

%% 首先，作者对比了几个之前的符号分析技术路线，他大致分成三类：
- symbolic-lifting frameworks.  e.g. Rosette, Chef.
- semantic frameworks. e.g. $\mathbb{K}$.
- multi-language IR-based tools. e.g. Viper, SAW, Infer. 
 %%
