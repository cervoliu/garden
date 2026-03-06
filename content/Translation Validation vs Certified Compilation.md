---
tags:
  - compilation
  - translation-validation
feishushare: true
feishu_url: "https://feishu.cn/docx/I4jrddTSdoQZsaxJ156cZeFBndc"
feishu_shared_at: "2026-03-06 13:13"
---

| **Technique**                 | **Translation Validation (TV)**                                                          | **Certified Compilation**                                                |
| ----------------------------- | ---------------------------------------------------------------------------------------- | ------------------------------------------------------------------------ |
| **Verification Target**       | Individual compilation results (input-output validation)                                 | Compiler implementation (process verification)                           |
| **Compiler Requirements**     | Can use existing off-the-shelf compilers                                                 | Typically requires building a new compiler from scratch (e.g., CompCert) |
| **Verification Scope**        | Per-compilation verification                                                             | Once-and-for-all compiler verification                                   |
| **Proof Generation**          | Generates proof witness for each compilation (bi-simulation relation or product program) | Compiler correctness proof is part of the compiler implementation        |
| **Key Challenge**             | Automatic identification of correlations between program transitions                     | Formal verification of complex compiler transformations                  |
| **Example Systems**           | Counter (Gupta et al. 2020), Alive2 (Lopes et al. 2021)                                  | CompCert (Leroy 2006), CompCertELF (Wang et al. 2020)                    |
| **Flexibility**               | Can validate any compiler's output                                                       | Tied to specific verified compiler                                       |
| **Verification Completeness** | Bounded validation possible (e.g., loop unrolling bounds)                                | Complete verification of compiler semantics                              |
| **Maintenance**               | Validation tool must evolve with compiler changes                                        | Compiler implementation and proofs must be maintained together           |
