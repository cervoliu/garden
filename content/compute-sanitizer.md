---
tags:
  - cuda
  - debug
---
CUDA 11.6 之前有个调试工具叫 `cuda-memcheck` ，11.6 弃用，并被 `compute-sanitizer` 取代。`compute-sanitizer` 是一个动态的检测工具，其原理是 dynamic instruction instrumentation。

Compute Sanitizer 中有四个主要工具：

- `memcheck`：用于内存访问错误和泄漏检测 (`--leak-check=full`)
- `racecheck`：共享内存数据访问危险检测工具
- `initcheck`：未初始化的设备全局内存访问检测工具
- `synccheck`：用于线程同步危险检测

使用样例见 [compute-sanitizer-samples](https://github.com/NVIDIA/compute-sanitizer-samples)
