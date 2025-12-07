#e-graph #term-rewrite

中文直译为等价图，最早提出于 Gregory Nelson’s [PhD Thesis](https://courses.cs.washington.edu/courses/cse599f/06sp/papers/NelsonThesis.pdf), 1980。

以下内容摘录自 egg 官网的 tutorial.

An _**e-graph_** is a data structure to maintain a [[Congruence Relation]] over expressions. An e-graph is a set of equivalence classes (**_e-classes_**), each of which contains equivalent _e-nodes_ (modulo rewrites and axioms). An **e-node** is an operator with children, but instead of children being other operators or values, **the children are e-classes**. In [[egg (e-graphs good) |egg]], these are represented by the [`EGraph`](https://docs.rs/egg/latest/egg/struct.EGraph.html "struct egg::EGraph"), [`EClass`](https://docs.rs/egg/latest/egg/struct.EClass.html "struct egg::EClass"), and [`Language`](https://docs.rs/egg/latest/egg/trait.Language.html "trait egg::Language") (e-nodes) types.

Even small e-graphs can represent a large number of expressions, exponential in the number of e-nodes. This compactness is what makes e-graphs a compelling data structure. We can define what it means to _represent_ (or _contain_) a term as follows:
- An e-graph represents a term if any of its e-classes do.
- An e-class represents a term if any of its e-nodes do.
- An e-node `f(n1, n2, ...)` represents a term `f(t1, t2, ...)` if e-class `ni` represents term `ti`.
Here are some e-graphs. We picture e-classes as dotted boxes surrounding the equivalent e-nodes. 

![[an e-graph undergoing a series of rewrites.png]]

An e-graph can be queried for patterns through a procedure called _e-matching_ (matching up to equivalence), which searches the e-graph for e-classes that represent terms that match a given pattern.

## Invariant

An e-graph has two core operations that modify the e-graph: [`add`](https://docs.rs/egg/latest/egg/struct.EGraph.html#method.add "method egg::EGraph::add") which adds e-nodes to the e-graph, and [`union`](https://docs.rs/egg/latest/egg/struct.EGraph.html#method.union "method egg::EGraph::union") which merges two e-classes. These operations maintains two key (related) invariants:

1. **Congruence**
    
    An e-graph maintains not just an [equivalence relation](https://en.wikipedia.org/wiki/Equivalence_relation) over expressions, but a [congruence relation](https://en.wikipedia.org/wiki/Congruence_relation). Congruence basically states that if _x_ is equivalent to _y_, _f(x)_ must be equivalent to _f(y)_. So as the user calls [`union`](https://docs.rs/egg/latest/egg/struct.EGraph.html#method.union "method egg::EGraph::union"), many e-classes other than the given two may need to merge to maintain congruence.
    
    For example, suppose terms _a + x_ and _a + y_ are represented in e-classes 1 and 2, respectively. At some later point, _x_ and _y_ become equivalent (perhaps the user called [`union`](https://docs.rs/egg/latest/egg/struct.EGraph.html#method.union "method egg::EGraph::union") on their containing e-classes). E-classes 1 and 2 must merge, because now the two “+” operators have equivalent arguments, making them equivalent.
    
2. **Uniqueness of e-nodes**
    
    There do not exist two distinct e-nodes with the same operators and equivalent children in the e-graph, either in the same e-class or different e-classes. This is maintained in part by the hashconsing performed by [`add`](https://docs.rs/egg/latest/egg/struct.EGraph.html#method.add "method egg::EGraph::add"), and by deduplication performed by [`union`](https://docs.rs/egg/latest/egg/struct.EGraph.html#method.union "method egg::EGraph::union") and [`rebuild`](https://docs.rs/egg/latest/egg/struct.EGraph.html#method.rebuild "method egg::EGraph::rebuild").
    