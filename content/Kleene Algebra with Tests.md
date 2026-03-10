---
aliases:
  - KAT
tags:
  - algebra
  - kleene-algebra
---
A Kleene Algebra with Tests (KAT) is $(\mathbb{A}, \mathbb{B}, +, ;, *, \neg, 1, 0)$, where
- $\mathbb{B} \subseteq \mathbb{A}$,
- $(\mathbb{A}, +, ;, *, 1, 0)$ forms a [[Kleene algebra]], and 
- $(\mathbb{B}, +, ;, \neg, 1, 0)$ forms a Boolean algebra

Example. Encoding programs

$$
\begin{aligned}
c;d  &\to c;d \\
\text{if} \; e \; \text{then} \; c \; \text{else} \; d \; \text{fi} &\to e; c+\neg e;d \\
\text{while}\; e \; \text{do} \; c \; \text{od} &\to (e;c)^{*};\neg e
\end{aligned}
$$
