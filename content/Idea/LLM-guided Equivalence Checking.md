---
draft: true
---
距离 Data-Driven Equivalence Checking 已经过去了十余年之久

既然 Data-Driven 能做，那 Deep Learning / LLM 为什么不能？

这一块似乎还没有人做，而且应该是个 low-hanging fruit

1. LLM-assisted Predicate Generation: 利用LLMs生成候选对齐谓词，然后通过本文提出的基于测试用例的PAA构建和验证流程进行筛选。LLM可以作为一个“智能的猜测器”。
2. Semantic Embedding for Program States: 训练深度学习模型将程序状态（寄存器值、内存状态等）映射到高维向量空间，使得语义上“对齐”的状态在向量空间中距离相近。对齐谓词可以在这个嵌入空间中被发现。
3. Graph Neural Networks (GNNs) for Control Flow Alignment: 将程序的控制流图（CFG）转换为图结构，利用GNNs学习两个CFG节点之间的语义对应关系，从而直接预测对齐路径或对齐谓词。
4. Reinforcement Learning for Search Strategy: 将对齐谓词的搜索过程建模为一个强化学习问题，LLM（或RL agent）根据PAA构建和验证的反馈（例如，是否成功验证、证明时间等），来学习更有效的谓词搜索策略。