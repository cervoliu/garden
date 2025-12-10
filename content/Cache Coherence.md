---
aliases:
  - 缓存一致性
  - 缓存连贯性
tags:
  - concurrency
---
考虑运行在多核 CPU 以及共享内存上的并发程序

| Time | Event                       | Cache contents for processor A | Cache contents for processor B | Memory contents for location X |
| ---- | --------------------------- | ------------------------------ | ------------------------------ | ------------------------------ |
| 0    |                             |                                |                                | 1                              |
| 1    | Processor A reads X         | 1                              |                                | 1                              |
| 2    | Processor B reads X         | 1                              | 1                              | 1                              |
| 3    | Processor A stores 0 into X | 0                              | 1                              | 0                              |
