---
aliases:
  - GPC
tags:
  - GPU
  - nvidia
  - architecture
---
GPC 是 Nvidia GPU 上真实存在的硬件微架构实体，典型层级如：
```
GPU
 └── GPC (多个)
      └── TPC / SM Group
           └── SM
                └── Warp / Thread
```

一个 **GPC 通常包含**：
- 若干个 **SM** 
- 共享的 **Raster Engine（在图形管线中）**
- 部分 **调度 / 前端 / 互连资源**
- GPC 内的 SM 具备低延迟通信与同步能力

Kepler 架构就已经有 GPC 了，但未通过 [[CUDA SIMT Programming Model|CUDA]] 暴露给程序员。直到 Hopper 架构，CUDA 才引入了 Thread Block Cluster 和 Distributed Shared Memory，允许程序员将 CUDA cluster 映射到同一个 GPC 或等价硬件域上
