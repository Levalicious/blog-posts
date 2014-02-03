# Introduction
The *linear assignment problem* (LAP) is a fundamental problem in the field combinatorial optimization. In it's most general form it can be described like this. There are $m$ workers and $n$ jobs with $m\leq n$. Assigning a worker to a job induces some cost. The task is to assign exactly one worker to each job in such a way that the total costs of the resulting assignment is minimized. Note, that in it's traditional formulation the number of workers is equal to the number of jobs (*squared* LAP with $m=n$). However, in a lot of applications the number of workers  is greater than the number of jobs (*rectangular* LAP with $m<n$).

The number of feasible solutions for arbitrary LAPs is $\prod_{i=0}^{m-1}(n-i)$. For squared LAPs the number of solutions is $n!$.  

A LAP can be represented by a cost matrix $C\in R^{m\times n}$ where each $c_{i,j}$ is the cost for assigning worker $i$ to job $j$. An assignment can be represented as a set of matrix element indices $\alpha=\{(i_1,j_1),\ldots,(i_m,j_m)\}$ in which no two distinct members have the same column index: $(r,s),(t,u)\in\alpha\Rightarrow r\neq t$ and $s\neq u$. The last property guarantees that every job has exactly one worker assigned.

There exist many efficient (polynomial time) algorithms to solve LAPs. The most famous, albeit not the most efficient of all of them is probably the *Hungarian method*. As you can find plenty of good explanations and implementations of the Hungarian method online, I won't describe it another time here. Instead I want to present an algorithm which uses the Hungarian method (or any other LAP solver) to find not only the very best assignment of a LAP but the first $k$-best assignments, where $k$ is a user-defined constant.

???? There's a big number of applications for a ranked set of linear assignments. A very prominent field is *multi-target tracking*. Beyond that, whenever integrating an additional constraint into the cost function is not feasible, finding a few of the best solutions ???

# Murty's algorithm
1968 Murty invented an algorithm for efficiently computing the first $k$-best solutions of a LAP. It's based on a classical method for solving LAPs; e.g. the Hungarian method. Murty's algorithm determines a ranked set of the $k$-best solutions of a LAP by repeatedly applying the LAP solver to a progressively adapted cost-matrix. The cost matrix is adapted in such a way that any feasible, already computed assignment is canceled out.

A priority queue of nodes is maintained. The nodes encode how do adapt the initial cost matrix. The cost matrices are adapted in such a way, that the previously calculated first $r$-best solutions ($r < k$) are excluded (impossible to find as a solution by the LAP solver). Initially, a new node with the initial cost matrix is added to the priority queue. Murty's algorithm progresses by repeatedly determining the node with the lowest costs from the priority queue. First, this node is added to the array of result assignments. Second, the node is partitioned and all new sub-nodes are inserted into the priority queue. Partitioning a node means generating restricted new nodes, encoding all possible sub-solutions of the node to be partitioned. Continuously canceling out cells of the cost-matrix allows at some point removing entire rows and columns from the matrix, effectively increasing the LAP solver performance.

I only want to give a general overview over the algorithm here. For a detailed derivation you can have a look into my master thesis.

complexity

# Summary

The Hungarian method is not the fastest, available LAP solver algorithm. In performance critical code the LAPJV (Jonker-Volgenant) algorithm (1987).