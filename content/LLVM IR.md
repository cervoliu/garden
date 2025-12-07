
知名开源编译工具链 [[LLVM]] 的 IR。

官网维护了非常丰富的 [language reference](https://llvm.org/docs/LangRef.html)

LLVM IR 实际上一种混合 IR，是线性 IR 与图 IR 的结合。

每个基本块用线性 [[Static Single-Assignment | SSA]] 指令列表表示，基本块之间用 CFG 表示控制流。

