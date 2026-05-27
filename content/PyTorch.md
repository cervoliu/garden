---
tags:
  - deep-learning
---
```
[用户编写的 nn.Module]
        │
        │  训练加速 / 推理加速
        ▼
┌──────────────────────────────────┐
│         torch.compile()          │  ← 统一入口
│                                  │
│  1. TorchDynamo (前端)           │  从 Python 字节码捕获 FX 图
│        │                         │
│  2. AOTAutograd                  │  提前展开前向 + 反向图
│        │                         │
│  3. TorchInductor (后端)         │  将图编译为 Triton/C++ 内核
│                                  │
│  (其他可选后端：ONNX, TensorRT)    │
└──────────────────────────────────┘
        │
        ▼
[高度优化的训练/推理内核执行]
```