---
tags:
  - deep-learning
draft: true
---
论文 [NIPS'22 FlashAttention: Fast and Memory-Efficient Exact Attention with IO-Awareness](https://dl.acm.org/doi/10.5555/3600270.3601459)

从标题可以得到的信息：
- 对比传统 Attention 的两个优点: Fast, Memory-efficient
- Exact Attention，计算的还是原来的那个 Attention
- with IO-Awareness，即优化通过改进 IO 的方式达成
