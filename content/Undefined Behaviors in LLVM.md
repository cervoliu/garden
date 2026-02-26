---
tags:
  - LLVM
  - undefined-behavior
  - compilation
aliases:
  - LLVM 中的未定义行为（Undefined Behaviors, UB）
zhihu-title: LLVM 中的未定义行为（Undefined Behaviors, UB）
zhihu-topics:
  - LLVM
  - 编译优化
  - 未定义行为
zhihu-link: https://zhuanlan.zhihu.com/p/2010283568631410795
zhihu-created-at: 2026-02-26 09:22
---
## Introduction

未定义行为 (UB) 用于描述我们不希望规定具体结果的 corner case 的行为。例如，我们可以将除以 0 的结果指定为 0，但由于我们并不真正关心结果，因此称之为 UB。

UB 也用于为 optimizer 提供额外的约束（例如，前端通过语言类型系统或运行时提供的保证）。

LLVM 中存在两种形式的 UB：immediate UB 和 deferred UB，其中 deferred UB 又包括 `poison` 和 `undef` 两种特殊值。此外，LLVM 还存在一种特殊的 `freeze` 指令用来阻止 `poison` 值的传播。

> The lattice of values in LLVM is: immediate UB > poison > undef > freeze(poison) > concrete value. （译注：实际上，这里的 > 可以理解成定义在 value 上的 refinement 关系，这是编译器优化所应当保证的，优化后程序与优化前程序之间满足的一种二元关系。）

接下来对这些值类别逐一介绍。
## Immediate UB

通常简称的「UB」 一般指的就是 immediate UB，是最强形式（最严重）的未定义行为，应尽可能避免。immediate UB 仅用来表示在 LLVM 后端支持的大多数 CPU 上都会触发 trap 的操作（例如除以 0，空指针解引用等）。在编译过程中遇到 immediate UB 一般会直接终止。

之所以应该避免 immediate UB，是因为它会使诸如提升（hoisting）之类的优化变得更加困难。请看以下示例：

```llvm
define i32 @f(i1 %c, i32 %v) {
  br i1 %c, label %then, label %else

then:
  %div = udiv i32 3, %v
  br label %ret

else:
  br label %ret

ret:
  %r = phi i32 [ %div, %then ], [ 0, %else ]
  ret i32 %r
}
```

我们可能会想通过移除分支并推测执行（speculative execution）除法来简化这个函数，因为 `%c` 大部分情况下都为真。这样我们就能得到以下 IR：

```llvm
define i32 @f(i1 %c, i32 %v) {
  %div = udiv i32 3, %v
  %r = select i1 %c, i32 %div, i32 0
  ret i32 %r
}
```

然而这个「优化」是不正确的！由于除数为 0 时除法会触发 UB，因此我们只能在**确定不会遇到这种情况时**进行推测执行。上面的函数 `f` 以 `f(false, 0)` 作为参数被调用时，优化前会返回 0，而优化后则会触发 UB。

这个例子突显了为什么 LLVM 在指定语义规范时要尽可能减少触发 immediate UB 的情况。一般来说，对于一条指令，只有当它大多数 LLVM 后端支持的 CPU 架构上都陷入 trap 时，才会将其语义定义为 immediate UB。

### Time Travel

LLVM IR 中的 immediate UB 允许所谓的「时间旅行」。这意味着，如果程序触发了 UB，编译器无需保留其任何可观察的行为，甚至包括 I/O。例如，以下函数在调用 printf 后会触发 UB（为什么？因为 LLVM 的 `willreturn` attribute 向编译器做出保证，即 `call` 指令的函数调用一定会返回。换句话说，被 `willreturn` 修饰的 `printf` 绝不会死循环，绝不会调用 `exit()` 终止进程，绝不会抛出异常从而跨越式地改变控制流。 `willreturn` 的背书使得编译器确信无论如何，控制流最终必然会回到 `@fn()` 并执行下一条指令，即 `unreachable`。然而， 执行 `unreachable` 在 LLVM 中是一个 immediate UB）：

```llvm
define void @fn() {
  call void @printf(...) willreturn
  unreachable
}
```

由于编译器知道 printf 函数总是会返回结果，并且 LLVM UB 可以进行时间旅行，因此可以完全移除对 printf 的调用，并将函数优化为：

```llvm
define void @fn() {
  unreachable
}
```
## Deferred UB

Deferred UB 是一种更轻微的未定义行为形式。它允许指令被推测执行，同时将某些 corner case 标记为具有特殊的，非正常的值。Deferred UB 适用于常见 CPU 提供的语义不同，但 CPU 不会触发 trap 的情况。

例如，考虑 shift 指令。当移位量大于等于位宽时，x86 和 ARM 架构提供的语义不同。我们可以通过两种方式解决这个问题：1）为 LLVM 选择 x86/ARM 中的一种语义，但这会导致为另一种架构生成的代码速度变慢；2）将这种情况定义为产生特殊值 `poison`。LLVM 选择了后一种方案。对于 C 或 C++ 等语言的前端（例如 clang），它们可以直接将源程序中的移位映射到 LLVM IR 中的移位，因为 C 和 C++ 的语义将此类移位也定义成 UB。但对于提供强语义的语言，它们必须有条件地使用移位值，就像这样：

```llvm
define i32 @x86_shift(i32 %a, i32 %b) {
  %mask = and i32 %b, 31
  %shift = shl i32 %a, %mask
  ret i32 %shift
}
```

LLVM Deferred UB 有两种特殊值：`undef` 和 `poison`，我们接下来将对其进行描述。

### `undef`

> Undef values are deprecated and should be used only when strictly necessary. **Uses of undef values should be restricted to representing loads of uninitialized memory.** This is the only part of the IR semantics that cannot be replaced with alternatives yet (work in ongoing).

`undef`: 表示某种特定类型的任意值。对同一个 `undef`的不同观测也可能会产生不同的值，例如：

```llvm
define i32 @fn() {
  %add = add i32 undef, 0
  %ret = add i32 %add, %add
  ret i32 %ret
}
```

不出所料，第一次加法运算的结果为 `undef`。然而，第二次加法运算的结果则更为微妙。你或许会认为它的结果是一个偶数，但事实并非如此。由于 `undef` 的每次（传递）使用都可能得到不同的值，因此第二次加法运算等价于 `add i32 undef, undef`，而这又等价于 `undef`。因此，上述函数等价于：

```llvm
define i32 @fn() {
  ret i32 undef
}
```

每次调用此函数都可能得到不同的值，即任意 32 位整数。

由于每次使用 `undef` 都可能得到不同的值，因此如果我们不能确定某个值是否为 `undef`，某些优化就可能失效。考虑一个将一个数乘以 2 的函数：

```llvm
define i32 @fn(i32 %v) {
  %mul2 = mul i32 %v, 2
  ret i32 %mul2
}
```

即使 `%v` 未定义，此函数也保证返回偶数。但是，正如我们上面看到的，以下函数则不然：

```llvm
define i32 @fn(i32 %v) {
  %mul2 = add i32 %v, %v
  ret i32 %mul2
}
```

正因为 `undef` 的存在，导致了上面的转换并不正确，即使在程序中都没有出现 `undef`，因为 LLVM 无法判断 `%v` 是否为 `undef`。

从本文最初的 value lattice 的角度看，`undef` 只能被 `freeze` 指令或具体值（concrete value）替换。因此，如果将 `undef` 作为操作数传递给某个指令，而该指令操作数的某些取值会触发 UB，则该程序本身也是 UB。例如，`udiv %x, undef` 是 UB，因为一旦我们将 `undef` 替换为 0（`udiv %x, 0`）就得到了一个 UB。

### `poison`

`poison` 是一种比 `undef` 更强的 deferred UB。它们仍然允许推测执行指令，但它们会被算数运算传播，进而污染整个表达式有向无环图（DAG）（有一些例外情况），类似于浮点数中的 `NaN`。

例子：

```llvm
define i32 @fn(i32 %a, i32 %b, i32 %c) {
  %add = add nsw i32 %a, %b
  %ret = add nsw i32 %add, %c
  ret i32 %ret
}
```

加法运算中的 `nsw` 属性意为，如果发生有符号溢出，则该运算会产生 `poison`。如果第一个加法运算溢出，则 `%add` 为 `poison`，因此 `%ret` 也为 `poison`，因为它污染了整个表达式 DAG。
Deferred UB 可以转化为 Immediate UB（branching on `undef` or `poison`）。

在编译优化中 `poison` 可被替换为任何类型的值（`undef`、具体值或 `freeze` 指令）。（译注：这大概是因为 immediate UB 并不是一种”值“的形式）

#### `poison` 经过 `select` 指令的传播

大多数指令只要有一个操作数是 `poison` 其结果就是 `poison`。一个值得注意的例外是 `select` 指令，当且仅当 cond 条件为 `poison` 或 selected value 为 `poison`时，其结果才是 `poison`。这意味着有时 `select` 指令起到了阻止 `poison` 传播的作用，从而影响到哪些优化操作可以执行。

例如，考虑以下函数：

```llvm
define i1 @fn(i32 %x, i32 %y) {
  %cmp1 = icmp ne i32 %x, 0
  %cmp2 = icmp ugt i32 %x, %y
  %and = select i1 %cmp1, i1 %cmp2, i1 false
  ret i1 %and
}
```

将 `select` 优化为 `and` 是不正确的，因为当 `%cmp1` 为 false 时，`select` 仅在 `%x` 为 `poison` 时才为 `poison`，而下面的 `and` 只要 `%x` 或 `%y` 为 `poison` 就为 `poison`：

```llvm
define i1 @fn(i32 %x, i32 %y) {
  %cmp1 = icmp ne i32 %x, 0
  %cmp2 = icmp ugt i32 %x, %y
  %and = and i1 %cmp1, %cmp2     ;; poison if %x or %y are poison
  ret i1 %and
}
```

但是，如果 `select` 的条件中使用了所有的操作数，则优化是正确的（请注意 `select` 指令中颠倒的操作数顺序）：

```llvm
define i1 @fn(i32 %x, i32 %y) {
  %cmp1 = icmp ne i32 %x, 0
  %cmp2 = icmp ugt i32 %x, %y
  %and = select i1 %cmp2, i1 %cmp1, i1 false
  ; ok to replace with:
  %and = and i1 %cmp1, %cmp2
  ret i1 %and
}
```

## `freeze` 指令

`undef` 和 `poison` 有时会在表达式 DAG 中过度传播。`undef` 是因为每次传递使用都可能观察到不同的值，而 `poison` 则会使整个 DAG 中毒。在某些情况下，阻止这种传播至关重要。这时就需要使用 `freeze` 指令。

以下函数就是一个例子：

```llvm
define i32 @fn(i32 %n, i1 %c) {
entry:
  br label %loop

loop:
  %i = phi i32 [ 0, %entry ], [ %i2, %loop.end ]
  %cond = icmp ule i32 %i, %n
  br i1 %cond, label %loop.cont, label %exit

loop.cont:
  br i1 %c, label %then, label %else

then:
  ...
  br label %loop.end

else:
  ...
  br label %loop.end

loop.end:
  %i2 = add i32 %i, 1
  br label %loop

exit:
  ...
}
```

假设我们要对上面的循环执行 loop unswitching 优化，因为循环内部的分支条件是循环不变的。我们将得到以下 IR：

```llvm
define i32 @fn(i32 %n, i1 %c) {
entry:
  br i1 %c, label %then, label %else

then:
  %i = phi i32 [ 0, %entry ], [ %i2, %then.cont ]
  %cond = icmp ule i32 %i, %n
  br i1 %cond, label %then.cont, label %exit

then.cont:
  ...
  %i2 = add i32 %i, 1
  br label %then

else:
  %i3 = phi i32 [ 0, %entry ], [ %i4, %else.cont ]
  %cond = icmp ule i32 %i3, %n
  br i1 %cond, label %else.cont, label %exit

else.cont:
  ...
  %i4 = add i32 %i3, 1
  br label %else

exit:
  ...
}
```

这里有个微妙的陷阱：当函数被调用时，如果 `%n` 为零，则原始函数不会在 `%c` 处进行分支，而优化后的函数会。由于 `%c` 可能为 `undef` 或 `poison`，而在 deferred UB 上进行分支会导致 immediate UB，因此这种转换通常是错误的。

这类情况需要一种方法来处理 deferred UB 特殊值，这正是 `freeze` 指令的作用所在。当传入一个具体值作为参数时，`freeze` 指令不执行任何操作，直接返回参数本身。当传入一个 `undef` 或 `poison` 时，`freeze` 指令返回一个该类型的非确定性值（non-deterministic value）。这与 `undef` 的不同之处在于 `freeze` 指令返回的值对所有用户都是相同的。

基于 `freeze` 函数返回值进行分支始终是安全的，因为它的值要么始终为 true，要么始终为 false。我们可以按如下方式修正上述 loop unswitching 优化：

```llvm
define i32 @fn(i32 %n, i1 %c) {
entry:
  %c2 = freeze i1 %c
  br i1 %c2, label %then, label %else
  ...
}
```
## 总结

LLVM IR 中的 UB 由两个明确定义的概念组成：immediate UB 和 deferred UB（`undef` 和 `poison` ）。将 deferred UB 传递给某些操作会导致 immediate UB。在某些情况下，可以使用 `freeze` 指令来避免这种情况。

LLVM 中的 value lattice 为：immediate UB > `poison` > `undef` > `freeze(poison)` > concrete value。值只能从左到右进行转换（例如，`poison` 值可以替换为 concrete value，但反之则不行）。

`undef` 值现已弃用，仅应用于表示未初始化内存的加载。

---
## 参考

- [LLVM Undefined Behavior 官方文档](https://llvm.org/docs/UndefinedBehavior.html)
- Juneyoung Lee, Yoonseung Kim, Youngju Song, Chung-Kil Hur, Sanjoy Das, David Majnemer, John Regehr, and Nuno P. Lopes. 2017. Taming undefined behavior in LLVM. In Proceedings of the 38th ACM SIGPLAN Conference on Programming Language Design and Implementation (PLDI 2017). Association for Computing Machinery, New York, NY, USA, 633–647. https://doi.org/10.1145/3062341.3062343
- Nuno P. Lopes, Juneyoung Lee, Chung-Kil Hur, Zhengyang Liu, and John Regehr. 2021. Alive2: bounded translation validation for LLVM. In Proceedings of the 42nd ACM SIGPLAN International Conference on Programming Language Design and Implementation (PLDI 2021). Association for Computing Machinery, New York, NY, USA, 65–79. https://doi.org/10.1145/3453483.3454030