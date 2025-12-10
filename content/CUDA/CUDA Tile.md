---
tags:
  - cuda
  - programming-model
  - GPU
---
CUDA Tile 于 2025 年 12 月随 CUDA 13.1 更新正式发布，NVIDIA 官方定性为 20 年来最大更新。官网介绍为:
> NVIDIA® CUDA® Tile is a tile-based GPU programming model that targets portability for NVIDIA Tensor Cores. CUDA Tile unlocks peak GPU performance with a programming model that simplifies the creation of optimized, ==tile-based kernels== across NVIDIA platforms.

支持 CUDA Tile 配套发布的有下层的 [CUDA Tile IR](https://docs.nvidia.com/cuda/tile-ir/)（编译 infra 是闭源的，因为涉及 NVIDIA GPU 硬件细节），以及面向用户的 [cuTile Python](https://github.com/NVIDIA/cutile-python)（以及未来的 cuTile C++）。CUDA Tile IR 是一组 Tile Programming 的虚拟指令集。

通过一张图大致了解一下 cuTile / Tile IR 在张量编译层次中所处的位置： 
![[Tile IR.png]]

可以看到，基于 Tile 的新编程范式与传统的[[CUDA SIMT Programming Model|SIMT 编程范式]]是平行的。说明英伟达也在重新平衡 performance 与 usability 的 tradeoff。

cuTile 在定位上与 [[Triton]]、[[tilelang]] 是比较重合的。