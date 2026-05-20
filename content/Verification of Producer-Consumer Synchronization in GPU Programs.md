---
tags:
  - PLDI15
  - paper-reading
  - verification
  - GPU
---
## 什么是 Warp-Specialized Kernel？ 

同一个 CTA 内的不同 warp 被分配执行不同的计算任务（通过基于 warp ID 的控制分支实现）。
这些 warp 之间通过命名屏障(named barriers) 进行生产者-消费者模式的同步,而非使用传统的 CTA 范围屏障。

论文用了一个直观的例子(Listing 1)说明 warp-specialized kernel 如何在不同 warp 间用命名屏障协调：

```c++
if (warp_id == 0) {
    bar.sync 0, 64;    // warp 0 在屏障0上等待
    bar.arrive 1, 64;  // warp 0 通知屏障1完成
} else {
    bar.sync 1, 64;    // warp 1 在屏障1上等待
    bar.arrive 0, 64;  // warp 1 通知屏障0完成
}
```

这里两个 warp 形成了一种信号传递模式。注意关键区别：`bar.sync` 是阻塞操作(warp 必须等待屏障完成)，而 `bar.arrive` 是非阻塞操作(warp 通知到达后立即继续执行)。正是这种非对称性使得生产者-消费者模式成为可能：生产者 warp 可以在用 arrive 通知数据就绪后,立刻继续下一轮生产,而不必等待消费者处理完毕。

## Warp-Specialized Kernel 的关键特征

总结起来,这类 kernel 有以下几个显著特征:

第一,异步的生产者-消费者同步。 不同 warp 扮演不同角色——有些充当选定的生产者,有些充当消费者。通过 arrive(非阻塞)和 sync(阻塞)的组合,可以实现生产者不等待消费者就能继续工作的流水线模式。

第二,使用命名屏障进行部分 warp 间的同步。 传统 syncthreads 要求 CTA 所有线程参与,而命名屏障允许只有部分 warp 参与同步,且每次使用可以指定不同的参与线程数。

第三,命名屏障是有限的硬件资源,需要被回收重用。 每个 SM 上只有 16 个物理命名屏障。每次屏障完成后,它会被立即重新初始化(即进入新的"世代 generation"),以便下一轮使用。论文在 Heptane 例子中特别展示了屏障 2 如何被安全地重用:只有当所有参与下一代的 warp 都与上一代建立了 happens-before 关系时,回收才是安全的。

第四,同步模式是静态的、编译期可确定的。 论文考察的所有 warp-specialized kernel 都使用直行代码(straight-line code),同步模式在编译时就能确定。这一特性是它们能被形式化验证(sound and complete)的关键前提。

