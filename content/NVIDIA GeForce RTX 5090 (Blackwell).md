---
tags:
  - GPU
  - hardware
  - nvidia
---
硬件参数：
- **32 GB GDDR7** 显存，采用 **512-bit** 总线，总带宽约 **1.79 TB/s**。
- **96 MB L2 Cache**（统一二级缓存，共享给所有 SM）。
- **170 个 SM（Streaming Multiprocessors）**。    
- 每个 SM 有 **128 KB** 的物理内存，默认作为 **L1 Cache**，可由 Kernel 分配为 shared memory 使用。