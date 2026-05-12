---
tags:
  - LLVM
  - concurrency
  - memory-model
---
The LLVM IR does not define any way to start parallel threads of execution or to register signal handlers. Nonetheless, there are platform-specific ways to create them, and we define LLVM IR's behavior in their presence. This model is inspired by the C++ memory model.

For a more informal introduction to this model, see the [[LLVM Atomic Instructions and Concurrency Guide]].

We define a *happens-before* partial order as the **least** partial order that
- Is a superset of single-thread program order ($\mathrm{po}$), and
- When $a$ *synchronizes-with* $b$ ($a \xrightarrow{\mathrm{sw}} b$), includes an edge from $a$ to $b$. *Synchronizes-with* pairs are introduced by platform-specific techniques, like pthread locks, thread creation, thread joining, etc., and by atomic instructions. (See also [[Atomic Memory Ordering Constraints]]).

Note that program order does not introduce *happens-before* edges between a thread and signals executing inside that thread.

Every (defined) read operation (load instructions, memcpy, atomic loads/read-modify-writes, etc.) $R$ reads a series of bytes written by (defined) write operations (store instructions, atomics stores/read-modify-writes, memcpy, etc.). For the purposes of this section, initialized globals are considered to have a write of the initializer which is atomic and happens before any other read or write of the memory in question. For each byte of a read $R$, $R_{byte}$ may see any write to the same byte, except: 
- If $write_{1}$ happens before $write_{2}$, and $write_{2}$ happens before $R_{byte}$, then $R_{byte}$ does not see $write_{1}$. (Coherence on a single memory location).
- If $R_{byte}$ happens before $write_{3}$, then $R_{byte}$ does not see $write_{3}$. 

Given that definition, $R_{byte}$ is defined as follows:
- If $R$ is volatile, the result is target-dependent. (Volatile is supposed to give guarantees which can support `sig_atomic_t` in C/C++, and may be used for accesses to addresses that do not behave like normal memory. It does not generally provide cross-thread synchronization.)
- TODO