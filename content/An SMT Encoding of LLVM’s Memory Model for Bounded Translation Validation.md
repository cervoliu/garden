---
tags:
  - paper-reading
  - SMT
  - memory-model
  - LLVM
  - CAV21
---
## Preface

LLVM Preliminaries:
- [[Undefined Behaviors in LLVM|Undefined Behaviors]]
- [[Reconciling High-Level Optimizations and Low-Level Code in LLVM#Data-Flow Provenance Tracking|Pointer Provenance]]

See also: [[Reconciling High-Level Optimizations and Low-Level Code in LLVM]]
## Motivating Example

```C
int f(int *p) {
	int *q = malloc(4);
	*q = 42;
	int *r = g(p+1);
	*r = 37;
	return *q;
}
```

```C
int f(int *p) {
	// q removed
	
	int *r = g(p+1);
	*r = 37;
	return 42;
}
```

