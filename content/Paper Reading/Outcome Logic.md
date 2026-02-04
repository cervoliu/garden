---
title: "Outcome Logic: A Unifying Foundation for Correctness andIncorrectness Reasoning"
tags:
  - paper-reading
  - program-logic
  - OOPSLA23
---
首先对比一下先前提出的两种经典的程序逻辑，Hoare Logic vs Incorrectness Logic

![[Program Logics in Triple.png]]
HL 是上近似逻辑，没有 false negatives (i.e. all executions of a verified program behave correctly)。这里，bug 对应 positive，bug-free 对应 negative
IL 是下近似逻辑，没有 false positive (i.e. any bug found using IL is in fact reachable by some execution of the program)

>[!ref]+
> IL achieves *true positives* (reachability of end-states) and *under-approximation* through the same mechanism: quantification over all states that satisfy the postcondition. However, this conflation of concepts leads to several problems:  
> 
> **Expressivity**. The semantics of IL only encompasses under-approximate types of incorrectness, which does not fully account for all bugs that may be encountered in real programs. For example, as we will see in Section 2.2, IL can be used to show the *reachability* of bad states, but it cannot prove *unreachability* of good states. 
> 
> **Generality**. IL is not amenable to probabilistic execution models and therefore is not a good fit for reasoning about incorrectness in randomized programs (Section 7.2).
> 
> **Error Reporting**. IL cannot easily describe what conditions are *sufficient* to trigger a bug (Section 6.6), meaning that analyses based on IL must implement extra algorithmic checks to determine whether a bug is worth reporting [Le et al. 2022].


指出问题：disjunction is not expressive enough