Checks for **refinement** between pair of LLVM IRs.

Designed to avoid false alarm (does not report spurious cex)

Fully models UB values ([[LLVM 中的未定义行为(UB)]])

---

![[Figure2.png|500]]

---

SMT encodings

- program register: (value, ispoison: bool)
	- the value could be integer(bit-vectors), float(SMT FPA theory), pointer(concat-bv of blockID and offset) or aggregate(concat-bv of each component)
- undef values: 
	- keep track of undef SMT variables used for each expression in the register file
- function argument %a: (ite(isundef_{%a}: bool, undef_{%a}, %a), ispoison_{%a}: bool)
	- undef_{%a}: fresh quantified variable to represent all the values of a type for **undef**
	- for aggregate, the above is encoded element-wise (enable represent each element to be **poison** or not)
	- however, still under-approximation -- only allow fully **undef** or not **undef** at all, don't allow partial **undef** (e.g. $\texttt{andi32 undef, 1}$)
- detecting **undef** (encoding of isundef) given expression $e$ with the set of undef variables $v$ : $e = e[k/v]$, where k is a constant and Alive2 use $k=3$
	- detecting if an expression can evaluate to more than one value
	- approximation of $\exists v,w. e\neq e[w/v]$
- return value: (value, no_return_condition: bool)
		- LLVM function can end with a call instruction to a function that does not return (e.g.,`exit`)
- program state S: (R, M, b: bool)
	- R is the register file
	- M is the memory
	- b is the UB flag
- final state: (r, M, ub: bool)
	- r is the return value
	- M is the memory at the return site
	- ub is the UB flag


==As LLVM’s IR is in SSA form, merge of values from different paths through the control-flow graph (CFG) is already explicit through the `phi` instruction. We merge the multiple  SMT expressions from the incoming paths of a basic block  trivially using the `phi` instructions, ending up with **a single SMT expression per register per function**. We **do not fork  expressions across paths** in the CFG.==

---

![[refinement definitions.png|600]]


![[refinement checking condition.png]]
Explanation:
1. Predicate $\text{valid}$ : encoding of global precondition (e.g. global variables should be assigned non-null and disjoint address)
2. $[\![f]\!](I)$: final state after executing function $f$ with input $I$
3. Predicate $\text{pre}$: encoding of precondition of a function
4. $N_{src}, N_{tgt}$: set of variables to encode non-determinism (i.e. **undef** values and **freeze** instruction) in function $f_{src}$ and  $f_{tgt}$

We check if: 
1. Any of the preconditions is always false; this can happen because of bugs or limitations in the encoding.  
2. The target triggers UB only when the source does.  
3. The return domain of the target is equal to that of the source, except for when the source triggers UB.  
4. The return value of the target is poison only when the source’s return value is poison.  
5. The return value of the target is undef only when the source’s return value is undef or poison.  
6. The return value of the source and target are equal when the source value is neither undef nor poison.  
7. Finally, if memory is refined.

---

Limitations:
- Bounded
- Intra-procedure, function-granularity verification
- **undef** value encoding is under-approximation -- do not allow partial **undef** on bit
- detecting undef value (isundef) encoding is under-approximation
- some tricky part of floating-point number modeling (do not support some type, some LLVM operation, and partial-support of NaN)
- Does not support many LLVM IR features: exceptions, function pointers, volatile variables, pointer-to-integer casts, type-based alias analysis, and many of LLVM's intrinsics