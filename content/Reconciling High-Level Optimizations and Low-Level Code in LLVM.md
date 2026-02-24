---
tags:
  - paper-reading
  - memory-model
  - LLVM
  - OOPSLA18
title: Reconciling High-Level Optimizations and Low-Level Code in LLVM
author: JuneYoung Lee et.al.
---
## Overview

应当是首篇正式地提出形式化的 LLVM memory model 的工作。

可以认为是后续 [[An SMT Encoding of LLVM’s Memory Model for Bounded Translation Validation]] 以及 [[Alive2 - Bounded Translation Validation for LLVM]] 的内存模型方面的理论基础。

> The two main questions a **memory model** needs to answer are (1) what is the return value of a load instruction, and (2) under what conditions is a memory-accessing instruction well-defined. A consequence is that the memory model should define which memory location a store instruction writes to.

## Prerequisites

- [[Undefined Behaviors in LLVM]]
---
## Memory Models for IR
### Flat Memory Model

最简单，最接近直觉的 memory model 被称为 *flat model（平坦内存模型）*。Flat model 认为内存不存在分块，所有的访存指令都在同一个大的内存数组上操作。

看一个简单的例子：

```C++
char *p = malloc(4);
char *q = malloc(4);
q[2] = 0;
p[6] = 1;
print(q[2]); // prints 0 or 1?
```

在 flat memory model 下，当`q == p + 4` 时，`printf` 打印结果为 `1`，否则为 `0`。

### Data-Flow Provenance Tracking

Flat memory model 下的程序行为高度依赖运行时（`malloc`），这种不确定性过度限制了编译器采取许多重要的编译优化，例如 store forwarding。事实上，在上面的那个例子中，自 C89 开始 C 语言标准就规定对 `p[6]` 的写操作不会 overwrite `q[2]` 处的值，不论 `p` 和 `q` 在运行时最终指向什么地址。

相较于 flat model，data-flow provenance 显得更“强”：

- 对于每个指针，记录 1. 其指向的对象（provenance，中文意为「起源」或「由来」）  2. 地址（或相对该对象的地址偏移） （解释：一般说指针 `*p` 指向对象 `a` ，我们说的是 `p == &a`，这里的意思其实是 `p ∈ [&a, &a + sizeof a)`）
- 指针的 out-of-bound 访问被视为 UB

同样用例子来理解：

```C++
char *p = malloc(4); // (val=0x10, obj=p)
char *q = malloc(4); // (val=0x14, obj=q)
char *q2 = q + 2; // (val=0x16, obj=q)
char *p6 = p + 6; // (val=0x16, obj=p) 
*q2 = 0; // OK
*p6 = 1; // UB, since out-of-bounds of obj p
print(*q2); // can be replaced with print(0);
```
### Extending Provenance to Integers

前一节的 data-flow provenance 不支持 int-to-pointer cast，因此本节的 memory model 将 provenance 推广到一般的 int 变量。

> In this model, each integer and pointer variable tracks a numeric value plus the object it refers to, or `nil` if none.

```C++
char *p = malloc(4); // (val=0x10, obj=p)
char *q = (int*)0x10; // (val=0x10, obj=nil) 

*q = 0; // UB, since obj=nil
if (p == q)
	*q = 1; // still UB; obj=nil 

int v = (int)p; // (val=0x10, obj=p)
int w = v + 2; // (val=0x12, obj=p) 

*(char*)w = 3; // OK 

char *r = malloc(4); // (val=0x14, obj=r)
int x = v + (int)r; // (val=0x24, obj=??)  data-flow provenance "break down"
int y = x - (int)r; // (val=0x10, obj=??)
```

但这自然会引入问题。第一个问题是， 并非所有的 int 变量都能赋予一个有意义的 data-flow provenance，例如最后两行的 `x` 和 `y`。

另一个致命的问题是，这个模型阻碍了太多的与整数有关的编译优化，例如通过 global value numbering (GVN) 或 range analysis 实现的等式传播。例如，在这个模型下将 `(a == b) ? a : b` 化简成 `b` 是不对的：即使 `a` 和 `b` 的 value 相等，它们的 provenance 不一定相等。再看一个例子：

```C++
char *p = malloc(4); // (val=0x10, obj=p)
char *q = malloc(4); // (val=0x14, obj=q)
int v = (int)p + 4; // (val=0x14, obj=p)
int w = (int)q; // (val=0x14, obj=q) 
if (v == w)
	*(int*)w = 2;  // not safe to replace this with *(int*)v = 2;
				   // which is routinely done by GVN
```

### Wildcard Provenance

这个内存模型把 int 变量的 provenance 又去掉了，但它与 [[#Data-Flow Provenance Tracking]] 不同的是，认为从 int 变量强制类型转换而来的指针的 provenance 可以是任意的对象（用 `*` 表示）。

```C++
char *p = malloc(4); // (val=0x10, obj=p) 
char *q = malloc(4); // (val=0x14, obj=q) 
int v = (int)p + 4; // (val=0x14) 
int w = (int)q; // (val=0x14)  
if (v == w) { 
	char *r = (int*)w; // (val=0x14, obj=*) 
	*r = 2; 
}
```

这样既支持了 int-to-pointer casting，又保证了 integer optimization 的合法。然而，`*` 的引入导致对含有 int-to-pointer cast 的程序做精确的别名分析异常困难。
### Inbounds Pointers

这一节作者介绍了截止论文发表（2018 年 11 月）时 LLVM 中正在采用的内存模型。

> In this section we explain the model currently used by LLVM where pointer arithmetic is optionally "inbounds", allowing some precision to be recovered by making out-of-bounds pointer arithmetic undefined:

这是什么意思呢？上一节我们提到，wildcard provenance 的引入会导致别名分析的困难，这是因为 wildcard 所代表的「任意的 provenance」太过宽泛了，导致别名分析的复杂度指数级上升。

LLVM的解决方案：引入“界内（inbounds）“属性，用于指针算术指令（`getelementptr`or `gep`）。当一个 `gep` 指令被标记为 inbounds 时，它施加了一个严格的规则：base pointer（operand）和 result pointer 都必须指向同一个内存对象（包括对象尾部的一个字节）。
如果这个规则被违反，那么 result pointer 就会变成 poison。poison 是 LLVM 中一种特殊，确定的未定义值，任何后续使用这个 poison 值的操作都会立即导致未定义行为（UB）。

如何恢复精度：通过将越界指针算术定义为 UB，编译器可以利用这些更强的保证来推断指针的有效范围和别名关系。如果编译器能证明某个指针算术操作会导致 poison，那么任何依赖于这个 poison值的后续内存访问都可以被视为永远不会发生（因为触发 UB）。

```C++
char *p = malloc(4); // (val=0x10, obj=p) 
char *q = foo(p); // (val=0x13, obj=p) 
char *r = q +inb 2; // poison: 0x15 is out of bounds of p  

p[1] = 0; 
*r = 1; // UB 
print(p[1]); // always prints 0
```

这种机制使得编译器能够对指针进行更激进的优化，例如在这个例子中，即使在运行时 q + 2 可能指向一个有效的内存地址，但由于它违反了 inbounds 的语义约定，编译器可以将其视为非法，从而允许进行更精确的别名分析。

## Memory Model for LLVM
