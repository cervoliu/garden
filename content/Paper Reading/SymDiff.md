---
aliases:
  - "SymDiff: A language-agnostic semantic diff tool  for imperative programs"
tags:
  - paper-reading
---
> SymDiff operates on programs in an intermediate verification language **Boogie**.

以 Boogie 作为验证目标语言

> SymDiff takes as input two **loop-free** Boogie programs and a configuration file that matches procedures, globals, and constants from the two programs. Loops, if present, can be unrolled up to a user-specified depth, or may be extracted as tail-recursive procedures.

做不了循环。