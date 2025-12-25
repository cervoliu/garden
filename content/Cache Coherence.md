---
aliases:
  - 缓存一致性
tags:
  - concurrency
  - architecture
---
Cache Coherence 是在具有 cache 的共享内存系统上，支持 [[Consistency model]] 的一种手段。由硬件提供 coherence 的支持。

考虑运行在多核 CPU 以及共享内存上的并发程序

| Time | Event                       | Cache contents for processor A | Cache contents for processor B | Memory contents for location X |
| ---- | --------------------------- | ------------------------------ | ------------------------------ | ------------------------------ |
| 0    |                             |                                |                                | 1                              |
| 1    | Processor A reads X         | 1                              |                                | 1                              |
| 2    | Processor B reads X         | 1                              | 1                              | 1                              |
| 3    | Processor A stores 0 into X | 0                              | 1                              | 0                              |
