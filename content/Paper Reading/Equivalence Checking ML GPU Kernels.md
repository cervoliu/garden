---
tags:
  - equivalence-checking
  - paper-reading
  - GPU
feishushare: true
feishu_url: "https://feishu.cn/docx/HVSWddTLaoqYwRxUPdQcVJdWneg"
feishu_shared_at: "2026-03-03 16:48"
---
## 概要

本 paper 来自 PLDI'26 审稿，目前挂在 arxiv 上暂未发表。

做 CUDA Kernel (CTA-scope) 的等价性验证：
- 我们知道一个 CUDA kernel 可以看成一个参数化函数 `Ker(blockId, threadId)`，其中 `Ker` 会有一些自定义的显式参数（例如用于 IO 的 tensor buffer），而 `blockId` 和 `threadId` 是类型为 `dim3` 的默认隐式参数。
	- 参数化函数 `Ker(blockId, threadId)` 作用于 grid 级别（规定 CUDA kernel 完整的执行）
	- 确定 `blockId` 的具体赋值，可以得到部分参数化函数 `Ker'(threadId)`，作用于 CTA 级别（规定每个加载到 SM 的 CTA 计算单元的执行）
	- 进一步确定 `threadId` 的具体赋值，则所有参数均被确定，得到作用于单个 thread 级别的函数（规定每个硬件线程的执行）
- some optimization may change tiling -- change input tensor sizes per CTA and number of CTAs per launch, yet preserve global equivalence -- which are beyond this work.
- since the CTAs of a kernel only differs in `blockIdx.x`, `blockIdx.y` and `blockIdx.z`, it suffices to verify the CTA instance pair with `blockIdx.x = blockIdx.y = blockIdx.z = 0`.

对 GPU Kernel 所做的核心假设：
- CTA 是「结构化」的（Structured-CTA）：
	- straight-line, i.e. no dynamic branches or loops (due to static-sized tensors and data independence)
	- static memory access, i.e. base array and index can be determined statically.
- tensor elements $\in \mathbb{R}$.

## Paper Review

### Paper summary

This paper presents an equivalence checker for machine-learning GPU kernels. The proposed tool takes as input two kernel programs: a reference implementation and an optimized version, which may be generated manually, by large language models, or through compiler optimization passes. The verification targets a restricted but practically motivated class of GPU programs, referred to as structured CTAs. These programs are straight-line, contain no dynamic branches or loops, and have statically determined memory accesses. The paper motivates these restrictions by arguing that they capture a large and practically relevant subset of real-world ML kernels. Furthermore, tensor elements are modeled over the real numbers rather than floating-point arithmetic, reflecting the observation that many kernel optimizations preserve mathematical equivalence while modifying floating-point evaluation details.

The equivalence checking algorithm is based on symbolic execution. The authors formally establish that the algorithm is sound and complete for the considered class of GPU kernel programs. Beyond equivalence checking, the approach can also soundly verify data-race freedom of kernels. The paper introduces a prototype implementation, called Volta, and evaluates it on a collection of GPU kernel benchmarks.

### Strengths

- Addresses the largely unexplored problem of equivalence checking for GPU kernel programs.
- Provides a clear and rigorous formalization of GPU kernel equivalence.
- Establishes soundness and completeness of the proposed approach for the considered program class.
- Presents a nontrivial decidability result for equality of multivariate real polynomials with exponentiation.
- The paper is well written, well structured, and easy to follow.

### Weaknesses

- The considered class of GPU programs, while well motivated, is fairly restrictive.
- Deadlock detection is claimed but not sufficiently explained or demonstrated.
- The experimental evaluation is limited in scope and lacks important details.

### Detailed comments for authors

Equivalence checking for CPU programs has been extensively studied, whereas equivalence checking for GPU programs remains relatively under-explored. It is therefore pleasing to see a submission that targets this important and practically relevant problem.

The paper focuses on a restricted class of GPU programs, termed *structured CTAs*. I have mixed feelings on this modeling choice. On the positive side, the restriction appears well motivated and reflects practical considerations observed in many real-world GPU kernels. On the negative side, the model is arguably too restrictive, as it limits attention to overly simplified kernels. That said, structured CTAs can reasonably be viewed as a starting point for studying equivalence checking of GPU kernel programs. I appreciate the authors’ efforts in carefully defining this model and providing motivation for the associated simplifications.

Under the structured-CTA assumption, programs are straight-line, contain no dynamic branches or loops, and have statically determined memory accesses. As a consequence, equivalence between two structured-CTA programs can be straightforwardly encoded as a symbolic formula, and symbolic execution can be naturally adopted to generate such formulas. The correctness properties established in Section 4.3 are important for validating the approach. However, given the restriction to straight-line programs, these results are not particularly surprising.

As a PLDI submission, the paper would benefit from a broader discussion of more complex GPU programming constructs. For example, warp-level primitives such as the `__shfl_*` instructions are widely used in practice to enable inter-thread communication and optimize performance. It would be valuable for the authors to discuss how the presence of such instructions would affect the verification framework, and in what aspects the approach would need to be extended or adapted. While I understand that supporting such primitives would significantly complicate verification, even a high-level discussion would strengthen the paper.

It also appears that the structured-CTA assumption plays a key role in enabling the use of SymEngine instead of a full SMT solver for deciding equivalence conditions. It would be helpful if the authors could clarify which assumptions in the structured-CTA model are critical for this choice, and under which relaxations an SMT solver would become necessary instead of SymEngine.

Furthermore, the introduction (line 74) claims that the proposed approach can detect both data races and deadlocks. While data-race detection is discussed in detail later in the paper, deadlock detection is not adequately addressed. It would be beneficial either to include a discussion of deadlock detection or to revise this claim accordingly.

The evaluation section is relatively weak compared to other parts of the paper. It reads more like a case study than a comprehensive evaluation. The benchmark set includes only a limited number of kernels (e.g., Reduction, GEMM, Attention, and a few others). Evaluating the tool on a broader range of GPU kernels would provide stronger empirical evidence for the effectiveness of the approach. In addition, important experimental details are missing from the data-race experiments, such as how the kernel programs were selected, the number of threads and synchronization operations in each kernel, whether the reported data races are real or spurious, and the verification time. 

**Minor Comments**

- On Page 3, references to “Listing 1a” and “Listing 1b” should be “Fig. 1a” and “Fig. 1b”, respectively.
- In Section 3.2, the paper states: “Later we define structured-CTAs that formally encode these assumptions (Section 4.1).” However, there're no callbacks to these assumptions when you defined the formal language $\mathcal{L}$.
- In Section 4, in the proof of Theorem 2, the phase “it grows monotonically easier to read/write g” is unclear, as “monotonically easier” is not defined.
- Why do you rely on *SymPy*'s simplification capabilities to perform operations like computing array offsets or evaluating branch predicates? Given that programs are structured-CTAs, shouldn't they be statically determined beforehand for each thread? 
- Since *Volta* consists of two main components -- a simulator and a decision procedure -- it would be more informative to report their runtimes separately in the evaluation.

### Questions for author response

1. How would the presence of `__shfl_*` instructions affect the proposed verification framework, and in what respects would the approach need to be extended or adapted?
2. Which assumptions in the structured-CTA model are critical for enabling the use of SymEngine, and under what relaxations would an SMT solver become necessary for deciding equivalence?
3. Does the proposed approach also support deadlock detection? If so, could the authors clarify how deadlocks are handled?
4. What are the reasons for selecting the current set of kernel programs for evaluation, and why were additional kernels not included?

---
后续思考：

- GPU Kernel 是 SPMD(Single Program Multiple Data) 的典型范式，是否可以利用/如何利用 block/thread-level symmetry 的性质？换句话说，Kernel 本身是 (blockId: dim3, threadId: dim3) 的参数化程序，能否借用参数化程序验证中的技术来验证 GPU Kernel？
- CUDA 的 Async primitive 愈发复杂，在越新的架构上越是如此。Async primitive 会带来怎样的 impact？