---
tags:
  - arxiv
  - paper-reading
---
## Implementation

TorchDynamo: 从 pytorch 函数中得到计算图。
EMerge 在 TorchDynamo 之上实现，从 Huggingface Transformer 以及 vLLM 中选取 model pair，交给 TorchDynamo 得到计算图，然后在计算图上尝试用 e-graph 导出等价关系。

### Input Matching

> Purely syntactic alignment by name or argument position is brittle in real systems due to inconsistent naming, layout differences (e.g., channels-first vs. channels-last), and input fusion (e.g., concatenated Q/K/V weights).

### Efficiency Optimizations

**Dependency-Based Blocking**: 为每个 e-class 计算前向可达输入集合。只有当两个集合相同时才有必要比较两个 candidate e-class
**Efficient Frontier Enumeration**: 对于每个待比较的结点，计算可达的 e-class 以剪枝共享边界的搜索。启发式搜索搜索顺序。

## Evaluation

**RQ1** Bug Detection：能不能找到现实场景中的实现 bug

baseline: TTrace
从 vLLM 和 Huggingface Transformer 的 issue tracker 上搜集了13个已被确认并修复的 bug

> For each bug, we reproduce the faulty behavior and construct a model pair that exercises the affected code path.

EMerge 能够检测 10/13 

**RQ2** Equivalence & Scalability：能否为工业级模型建立等价关系

benchmark 覆盖了 GPT2，Qwen，Llama，Mistral，Phi，5 个开源大模型。


**RQ3** Rule Characterization：EMerge 能自动合成的什么样的重写规则
- baseline: Entangle (ASPLOS'26)