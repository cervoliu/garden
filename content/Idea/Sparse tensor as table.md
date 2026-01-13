可能相关： 
- [LingoDB on Github](https://github.com/lingo-db/lingo-db)
- [Subtrait on Github](https://github.com/substrait-io/substrait)
- [Galley: Modern Query Optimization for Sparse Tensor Programs](https://dl.acm.org/doi/pdf/10.1145/3725301)

> **tensor / 计算图 ↔ 关系代数 / 查询计划**

## 第一层直觉：sparse tensor ≈ relational table

如果我们把一个 **sparse tensor** 写成最“原始”的形式：

> 一个 tensor = 一组

> **(index₁, index₂, …, indexₖ, value)**

> 的集合

那它几乎就是一张关系表：

```
Tensor(i, j, k, v)
```

- **index 维度** → 关系表的 key / attribute
- **value** → payload
- 稀疏性 = 表中只存非零行

---

## **计算图中的 tensor op ↔ 关系代数算子**

下面是关键：**tensor 运算 ≈ 关系代数 + 聚合**

### **(1) Elementwise ops**

```
C = A + B
```

对应关系代数：

```SQL
SELECT i, j, A.v + B.v
FROM A JOIN B USING (i, j)
```

👉 本质是 **equi-join + map**

---

### **(2) Reduction / contraction（核心）**

经典例子：

```
C[i, k] = Σ_j A[i, j] * B[j, k]
```

关系表达：

```SQL
SELECT A.i, B.k, SUM(A.v * B.v)
FROM A JOIN B ON A.j = B.j
GROUP BY A.i, B.k
```

可以看到一个**非常干净的映射**：

|**Tensor**|**Database**|
|---|---|
|index match|join|
|multiply|expression|
|reduction|group-by + aggregate|

> **tensor contraction = join + aggregation**

---

### **(3) Broadcasting / reshape**

- reshape → schema transform
    
- broadcast → implicit replication / lateral join
    
- transpose → attribute renaming + reordering

在关系模型里都可以解释，只是有点“丑”。

---
## **3️⃣ 计算图 vs 查询计划：结构级对应**

### **Tensor 计算图**
- node = op
- edge = tensor
- DAG    
- 数据流为主
### **数据库查询计划**
- node = operator (scan / join / agg)
- edge = intermediate relation
- DAG 
- 控制流弱、数据流强

> **两者在 IR 层级几乎是同构的**

你甚至可以画一个一一对应的 mapping：

| **ML / Tensor**                  | **DB**                        |
| -------------------------------- | ----------------------------- |
| computation graph                | query plan                    |
| operator fusion                  | operator pushdown             |
| common subexpression elimination | view reuse                    |
| cost model (FLOPs / memory)      | cost model (IO / cardinality) |

---
## **4️⃣ 已有研究：这不是空想**

你这个问题，其实已经形成了一个清晰的研究谱系。
### **(1) Tensor Algebra ↔ Relational Algebra**

代表工作：
- **Relational Algebra for Arrays**
- **Generalized Linear Algebra (GLA)**
- **TACO / TVM** 的理论基础之一
    
核心观点：

> Join + Group-by 是 tensor contraction 的通用执行模型

---
### **(2) 数据库系统做 ML**

典型系统：
- **TensorFlow Fold / TensorFlow XLA**
- **MADlib**
- **SystemML**
- **Flink / Spark ML**    
- **Relational Deep Learning (RDL)**

它们都在做：  

> 把 tensor 运算编译成 relational-style execution

---
### **(3) 数据库优化技术反向用于 tensor**

这是你可能**最感兴趣**的方向：

|**DB 技术**|**Tensor 世界**|
|---|---|
|join reordering|contraction reordering|
|cost-based optimizer|kernel selection|
|predicate pushdown|sparsity-aware pruning|
|materialized view|cached intermediate tensor|
👉 MLIR / TVM / XLA 都在**偷偷做数据库的事**

---
## **5️⃣ 关键分歧点（非常重要）**

到这里要说清楚：**不是所有东西都能无损对齐**

### **❌ 不完全等价的地方**

#### **(1) 数据模型**

- DB：**set / multiset**   
- Tensor：**ordered, dense-friendly, shape-sensitive**

shape 在 DB 里是“弱约束”，在 tensor 里是“硬约束”。

---
#### **(2) 执行假设**
- DB：IO-bound、延迟隐藏、pull-based
- Tensor：compute-bound、SIMD/GPU、push-based
    
所以：

> 同一个“join + agg”，在 GPU tensor 上的最优实现，和 DB 上完全不同

---

#### **(3) 代数封闭性**

Tensor contraction 通常在 **semiring** 上：

- (+, ×)
    
- (max, +)
    
- (or, and)


数据库聚合不是天然支持所有 semiring（但可以扩展）。

---

## **6️⃣ 一个“干净”的统一视角（推荐）**
  
> **Tensor computation = Evaluation of an expression in a semiring over a multi-relational structure**

  

- 稀疏 tensor → relation
    
- contraction → join
    
- reduction → aggregate
- 
    
- autodiff → provenance / lineage
    
这在理论上是非常漂亮的。