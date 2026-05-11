---
tags:
  - PLDI26
  - LLVM
  - undefined-behavior
---


LLVM IR 中，C/C++ 中的库函数调用 `memcpy` 一般通过 intrinsic call（LLVM 内置函数，具有固定语义）来表示：

```llvm
; copies 4 bytes
call void @llvm.memcpy(ptr %dst, ptr %src, i64 4, i1 false)
```

然而，编译优化通常会把 intrinsics 视为黑盒，这阻碍了许多优化的分析，因此 LLVM 对  `memcpy` 有一条针对性的优化 pass，当拷贝内存块长度较小时，它会将 `memcpy` 展开成逐字节 `load`/`store` 的序列，以暴露出潜在的优化机会：

```llvm
%v = load i32, %src
store i32 %v, ptr %dst
```

然而，这层转换是不 sound 的！原因有两点：
1. 这两条 `load`/`store` 以 `i32` 整数类型来拷贝内存值，如果某块地址处是指针类型，那么会发生隐式的 ptrtoint 和 inttoptr 类型转换，导致 pointer provenance 丢失。 
2. 当只有某个 bit 处的值为 `poison` 时，用 `i32` 会导致 `poison` 扩散至整个 `i32` 的 32 个 bit，拷贝前后的值不一致（回忆，`poison` 对于 memory 来说是 bit-wise property，但对于 register 是 value-wise property）。



