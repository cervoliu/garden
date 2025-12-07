---
tags:
  - tensor
  - MAPS2021
  - paper-reading
---
## motivation

现有表示 ML 计算的 IR 要么 pure but high-level，无法表达 low-level 的针对目标硬件的 term rewrite。要么 low-level 但是 impure，impurity 阻碍了 term rewrite（ IR 的有状态性使得我们不能像对待数学表达式一样去做等式替换 ）。本文提出一种 pure 的 Glenside IR，通过针对访问的抽象 **access pattern** 实现 low-level, layout-aware, hardware-centric program rewrite。

## motivating example

用 `f64` 表示 tensor type 64-bits float，用 `[A]` 表示类型为 A 的 vector

采用 functional programming 的表示，`f: p1 * p2 * ... * pn -> r` 使用参数类型列表和返回值类型定义一个函数对象

内积 `dotProd : [f64] * [f64] -> f64`
2D转置 `trans2 : [[f64]] -> [[f64]]`
对向量的映射 `map: (A -> B) * [A] -> [B]`，将一个签名为 `(A -> B)` 的函数映射到向量 `[A]` ，得到向量 `[B]`
笛卡尔积 `cartProd : [A] * [B] -> [A * B]`，输入参数类型为 `[A]`和 `[B]`，返回一个类型为 `A * B` 的 vector，包含所有由 `[A]` 与 `[B]` 中元素组成的 pair （pair 的类型为 `A * B`）。当 `A = B = [f64]` 时，其返回类型刚好符合 `dotProd` 的签名。应该与[[tensor outer product | 张量外积]] 是同一个运算。
矩阵乘 `matMul(P,Q) := map(dotProd, cartProd(P, trans2(Q)))`，即将矩阵乘视为行向量与列向量的内积，那么这里的 `P` 与 `Q` 的类型都是 `[[f64]]`。

但是这个定义下的 `matMul` 会将返回类型 flatten 成 `[f64]`，不是我们期待的 `[[f64]]`。究其原因，在于 `cartProd` “忘记”了参数的 shape。

重新定义一个笛卡尔积
`cartProd2D : [A] * [B] -> [[A * B]]`（这里的简单类型并没有指明返回的矩阵是怎么排布 `A * B` 类型元素的，文章中简单假设了排布方式满足矩阵乘法的 specification）

以及 `mapAt2 : (A -> B) * [[A]] -> [[B]]` 

可以更改矩阵乘的定义为 ```
```
matMul(P, Q) := mapAt2(dotProd, cartProd2D(P, trans2(Q)))
```

但这样定义的矩阵乘要求定义 dimension-specific operators（`mapAt2`, `cartProd2D`），没有维度抽象。如果用这种 IR 表示一般情况（高维度）下的 rewrite rule，会发现：
- we have to specify all the variants of tensor kernels (at different dimensions)
- one rewrite rule -> multiple versions of every combinations of dimensions

一种解决方式是用 lambdas, currying and closures 来表示:
```
matMul' P Q :=
	map' (λ r => map' (dotProd' r) (trans2 Q)) P
```
这里的 `matMul'`, `map'`, `dotProd'` 都是 curried operator

或者用 index notation 表示
```
matMul(P,Q)[i,j] := dotProd(P[i], trans2(Q)[j])
```

但这两种表示仍然依赖某种 _name binding_（前者是 r，后者是 i, j ）。被重写的 expression 如果包含 binded-variable，就不能简单地做模式匹配与替换，而需要分析 _contexts_ (what names are bound to)，增加额外的复杂度。

Glenside IR 的目标: 不依赖于 name binding, supports specifying and composing higher-order tensor operators over arbitrary dimensions
## Glenside

### Access Patterns

Observation: some tensor dimensions are _iterated over_ (accessed) while others are _computed on_. 

encode such common tensor IR patterns by their _shape_ -- a pair of tuples of positive integers $(S_A, S_C)$. 


![[Glenside’s access pattern transformers.png]]
（勘误：windows 行 Output Shape 列，最后 $b_i' = \lceil (b_i - (w_i - 1)) / s_i \rceil$）

![[Glenside’s access pattern operators.png]]

## Case Studies
### Mapping `matMul` to Accelerators

脉动阵列（Systolic Array）特性: 
- 脉动阵列是一种常见的矩阵乘法硬件架构。一个 r 行 c 列的权重固定脉动阵列，接受两个输入：一个长度为 r 的向量列表（通常是激活值 activations），另一个长度为 c 的向量列表（通常是权重 weights）。它将第一个列表中的每个向量与第二个列表中的每个向量进行配对，并计算每对向量的点积。
Glenside 的表示: 
- 在 Glenside 中，矩阵乘法通常表示为 `(compute dotProd (cartProd ?a0 ?a1))`，这表示对 `?a0` 和 `?a1` 这两个访问模式的笛卡尔积（cartProd）的结果执行点积计算。
重写规则: 论文中展示的重写规则如下：
```
(compute dotProd (cartProd ?a0 ?a1)) =>
(systolicArray ?rows ?cols ?a0 (access (transpose ?a1 (list 1 0)) 0))
where ?a0 is of shape ((?batch), (?rows))
and ?a1 is of shape ((?cols), (?rows))
```

左侧 (LHS): `(compute dotProd (cartProd ?a0 ?a1))` 匹配任何进行点积计算的笛卡尔积模式。
`?a0` 和 `?a1` 是模式变量，分别绑定到输入访问模式。

右侧 (RHS): `(systolicArray ?rows ?cols ?a0 (access (transpose ?a1 (list 1 0)) 0))` 是重写后的结果，它引入了一个新的 `systolicArray` construct 来表示对硬件的调用。

`?rows` 和 `?cols` 参数定义了脉动阵列的大小。
`?a0` 直接作为脉动阵列的一个输入。
`?a1` 经过了转换：`(access (transpose ?a1 (list 1 0)) 0)`
`transpose ?a1 (list 1 0)`: 将 `?a1` 进行转置。
`access ... 0`: 将转置后的 `?a1` 视为一个访问模式，其所有维度都作为访问维度（access dimension），计算维度为空（这通常意味着整个张量被视为一个整体进行访问）。

这种转换是为了更准确地模拟实际脉动阵列硬件访问权重张量的方式：它一次性读取整个张量，并期望它以转置的形式布局在内存中。

条件 (Condition): `where ?a0 is of shape ((?batch), (?rows)) and ?a1 is of shape ((?cols), (?rows))` 确保只有当输入访问模式的形状符合脉动阵列的要求时，才应用此重写规则。

益处: 这种方式通过 Glenside 的访问模式，能够提供更丰富的数据布局信息，这对于后续的重写或代码生成步骤非常有帮助，因为它允许编译器在更高层次上理解和操作硬件特定的数据访问需求。

### Flexible Mapping: Discovering [[im2col]]
