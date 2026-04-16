---
tags:
  - verification
---
## LLVM IR

### SeaHorn

![[SeaHorn.png]]

前端：C/C++/... --> LLVM bitcode
中端验证条件生成：LLVM bitcode --> CHC
后端 Solver：Spacer
实现语言：C++
维护状态：上次更新于 2 年前

### KLEE

仅符号执行框架，不包含验证部分
实现语言：C++
维护状态：上次更新于 2 个月前
### SMACK

前/中/后：C --> LLVM bitcode --> Boogie intermediate verification language --> Boogie/Corral Solver
维护状态：上次更新于 5 年前
## Binary

### Triton

![[Triton.png]]
前端：二进制文件
中端：Triton AST，可以 lift 至 LLVM IR
后端：Solver Interface，支持 Z3, Bitwuzla
实现语言：C++
维护状态：上次更新于 6 个月前
### BINSEC

实现语言：OCaml
维护状态：仍处于早期版本（before v1.0），目前积极维护中