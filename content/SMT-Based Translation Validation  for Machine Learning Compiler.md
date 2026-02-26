---
tags:
  - CAV22
  - translation-validation
  - MLIR
  - tensor
---
## 总结

- 本文最主要的贡献在于提出了一套针对 ML 编译中的各种对象（张量，浮点数，张量运算/循环，内存）的 SMT 编码的解决方案。这套 SMT-based 的建模在精确度和效率之间存在很多取舍：
    
    - 提供了 3 个不同的用于验证的抽象层次
        
        > To make validations cheap on average, we progressively refine the abstraction scheme that describes the abstraction level of encodings.
        
    - Abstract floating point encoding
        
    - Hash-based encoding of multiset objects (出现于 reduction 运算)
        
- Artifact: the `mlir-tv` tool. Supports 25 `tosa` ops, 11 `memref` ops, 13 `linalg` ops, 10 `tensor` ops, 29 `arith` ops, 3 `bufferization` ops, and 8 other ops.
    

## SMT Encoding

- Tensor: SMT array。具体地，是一个从 address space-sized bitvector 到 element type 的映射
    
    - domain type 是固定位数的 bitvector
        
        - 这里其实有一个问题：bitvector 本身描述的是**向量**的 domain type，不是张量的 domain type
            
        - 实际上，这里是将(多维)张量看(建模)成以列主序存储的一维向量
            
    - element type 是前面提到的 abstract floating point
        
    - 每个 tensor 除了表示值的 array，还附加一个表示是否被初始化的 bool array。将读取未初始化的张量元素定义为 UB。
        

```Lisp
; 0. 假设定义了某种抽象的浮点类型，按抽象层次，可能是 UF 或者更具体的浮点建模
(declare-sort AbstractFPType 0)
; "address space-sized bit-vector"
; 假设地址空间宽度为 32 位
(define-sort AddrSort () (_ BitVec 32))

; 1. 定义张量 A
(declare-const A (Array AddrSort AbstractFPType))
(declare-const A_init (Array AddrSort Bool))

; 2. 定义动态形状 (Dynamic Shapes)
; Dimension sizes are encoded as bit-vector variables
(declare-const dim0 AddrSort) ; 行数 (Height)
(declare-const dim1 AddrSort) ; 列数 (Width)

; 3. 具体的运行时大小 (在这个例子中假设是 2x3)
(assert (= dim0 #x0002))
(assert (= dim1 #x0003))

; 4. 大小约束断言
; 确保 dim0 * dim1 没有溢出且在地址空间范围内
(assert (bvule (bvmul dim0 dim1) #xFFFF))
```

- Tensor arguments: fresh SMT array variable
    
- Tensor ops:
    
    - 对于一般的算子，可以精确建模成 lambda expression
        
    
    > For example, a negation of tensor t is encoded as `lambda i, negate(select(t, i))`
    
    - 对于 Reduction ops，用 UF 做上近似，这一点和 [TensorRight: Automatic Verification of tensor graph rewrite.pdf](https://xfayy0htye.feishu.cn/wiki/ZeCsw31k2iLFXAkzSnDc7zdHnxf) 的做法类似
        
    
    > In general, reduction operations like summation of an array cannot be precisely encoded in SMT-LIB 2. To support them, **we abstractly encode the reduction operations using UFs**. For example, we declare `sum`which is a UF taking an array and returning a float number.
    
- `linalg` loops:
    
    > `linalg` loop 是 MLIR 中一种高级的、结构化的循环操作，通常由更高级别的张量操作转换而来。它们比一般程序中的循环更简单，并提供丰富的语法信息，例如明确的输入/输出张量索引映射（index mapping）以及可并行化的归纳变量（parallelizable induction variables）。循环体通常不包含副作用（除了可能触发未定义行为）。这些特性使得在 SMT 中编码 `linalg` 循环时，无需推断复杂的循环不变量，可以直接构建输出张量。
    
    - 使用 lambda theory + universal quantification 来编码
        
    - 具体编码步骤：
        
        - 通过 index mapping 以及张量对象（或内存缓冲区对象）的大小确定循环的上下界
            
        - 编码循环体
            
        - 如果输出为张量对象，则结果张量被编码为 lambda 表达式。该 lambda 表达式将循环的归纳变量作为参数，并根据行主序计算出张量中每个元素的值。例如，一个双重嵌套循环的结果可能被表示为 `lambda (d0, d1), expr`。
            
    
    ```Assembly
    #id = affine_map<(d0, d1) -> (d0, d1)> 
    #transposed = affine_map<(d0, d1) -> (d1, d0)>  
    // %C = %A + %B^T, %C’s shape = %out’s shape  
    %C = linalg.generic {
            indexing_maps = [#id, #transposed, #id],
            iterator_types = ["parallel", "parallel"]} 
          ins(%A, %B : tensor<?x?xf32>) outs(%out : tensor<?x?xf32>) { 
     ^bb0(%a: f32, %b: f32, %unused: f32): 
      %c = arith.addf %a, %b: f32 
      linalg.yield %c : f32 
    } -> tensor<?x?xf32>
    ```
    
- Reduction 运算的代数性质
    
    - 交换律(commutativity) : SMT multiset theory 效率低下，用 hash-based bv encoding 近似
        
- Final states: `(ub: bool, m, v)` where `ub` is UB, `m` is the memory, and `v` is the return value.
    
- Refinement of final states $$(ub, m, v) \sqsupseteq (ub', m', v')$$: (1)$$ub$$ is true, or (2) $$ub=ub' \land v=v' \land m \sqsupseteq m'$$
    
    - Refinement of memory $$m \sqsupseteq m'$$: if for non-local blocks `(b, b')` with same id in the source and target, if (1) reading `b` at offset `o` is successful, so does the access to `o` at `b'`, and (2) if `b` is writable, so does `b'`.
        
- Refinement of functions: For any input state $$I$$ consisting of an initial memory and argument values, $$f_{src}(I) \sqsupseteq f_{tgt}(I)$$ must hold where $$f(I)$$ denotes the final state of function $$f$$ .
    

## 实验评估

### MLIR Unit Tests

- **449 tests**, containing **2467 function pairs** in total
    
- The unit tests (1) apply specific transformations to small, pre-defined MLIR programs, and (2) check whether the output programs syntactically match the test patterns.
    
- Using `mlir-tv` to validate transformations preserve semantic
    
- Find **real semantic issues** of MLIR dialects
    

### Validating Compilation of Deep Learning Models

- 三个 model: `text_classification_v2`, `SqueezeNet` and `MobileNet`，分别用 MLIR 的这些编译趟： `tosa-to-linalg`, `tosa-to-standard`, `canonicalize`, `fuse-elementwise-ops`, `tensor-constant-bufferize`, `linalg-bufferize`, `tensor-bufferize` 编译。用 `alive-tv` 验证每一趟。
    
- To validate them in a reasonable time, we split the source and target programs into smaller functions. （手动做了一些处理）
    

## Related Work

分成 4 部分来讲，分别是 Verifying Programs with Floating-Points，Verifying Programs Using Arrays，Machine Learning Compilers，Compiler Verification.

> **Compiler Verification.** [11] relaxes FPA semantics since a compiler can ignore strict IEEE-754 behavior like fast-math optimizations in LLVM. They propose Icing which is a language allowing IEEE 754-unsafe FPA optimizations, and CakeML [28] which is a verified compiler with the optimizations. [32] proposes a verified tensor optimizer whose optimizations can be explored via Coq’s tactics. As for translation validation (TV), [18] proposes a practical TV framework for Halide which is a language for processing arrays. To support fast-math optimizations, it mainly uses Z3’s type for real numbers. For general-purpose compilers, many different tools have been developed [36]. **Alive2** [33], LLVM-MD [42] and Peggy [41] validate the transformations in LLVM using various techniques. **The SMT memory model for Alive2 [30] uses a technique that is similar to our approach in order to bound the number of memory blocks.** Some TV tools [17,22] split the original programs and validate the smaller pairs.

  

## Artifact

https://github.com/aqjune/mlir-tv