# CMU SCS 15-799 (Spring 2025): Join Ordering: Top-Down

## 1. Introduction & Administrivia

This lecture continues our discussion on **join order enumeration**, focusing specifically on **top-down** approaches. 

---

## 2. Recap: DPHyp (Hypergraph Enumeration) [from Lecture 7]

- **DPHyp** enumerates connected subgraphs, incrementally adding hyperedges.  
- Hyperedges allow considering multi-node join predicates as a single expansion.  
- Good for complex queries, but the approach is still predominantly **bottom-up**.

---

## 3. Motivation for Top-Down Approaches

**Why Top-Down?**  
- **Demand-driven** expansions and transformation rules.  
- **Branch-and-Bound** pruning.  
- **Partial plan exploitation** (memoization).  

However, top-down search historically had no general solution guaranteeing no cross-product joins without enumerating exhaustively. That’s what the **Partition-based Top-Down** method aims to solve.

---

## 4. Optimal Top-Down Partitioning (OTDP)

### 4.1 Overview

A new approach (SIGMOD 2007 paper) to **recursively split the query’s join graph** into smaller partitions and generate join orders from these partitions.  
- **Partition** the graph into subgraphs.  
- Recursively handle each subgraph, then combine results.  
- The *quality* of the final plan heavily depends on **which edges or vertices** we cut.

**Key point**: “Optimality” here refers to the **algorithm’s efficiency** in enumerating valid plans, **not** necessarily the cost of the final query plan.

### 4.2 Graph Complexity

We measure complexity not just by the number of relations, but by the **structure** of the join graph.  
- E.g., chain vs. clique vs. star shape.  
- Partition-based top-down enumerations must consider **connectedness** after cuts to avoid unintended Cartesian products.

### 4.3 Graph Encoding

1. **Edge-List**: Keep a list of vertex pairs (edges).  
2. **Bit-Array**: For each vertex, store a bitmap indicating connections.  
   - Allows fast bitwise operations (intersection, union, etc.).  
   - Minimizes overhead in repeated set operations during enumeration.

---

## 5. Naïve Partitioning Algorithms

1. **Left-Deep w/ Cartesian Products**  
   - Remove each vertex one at a time.  
   - Causes undesirable cross-joins.

2. **Left-Deep w/o Cartesian Products**  
   - Check connectivity after removing a vertex.  
   - Prune if removing the vertex disconnects subgraphs.

3. **Bushy Plans w/ or w/o Cartesian Products**  
   - Partition based on all non-empty subsets of vertices.  
   - Again, check connectivity to avoid cross-joins.

**Issue**: None of these exploit the **graph structure** effectively; they just check connectivity post-hoc.

---

## 6. Min-Cut Partitioning

A more refined method for partitioning a join graph:
- **Generate a biconnection tree** from a randomly chosen vertex.  
- Identify the minimal set of edges whose removal **partitions** the graph into subgraphs without losing connectivity.  
- Depth-first exploration on this tree to produce partitions.  

**Benefit**: Preemptively identifies “bad edges” that would cause cross-joins or isolate nodes incorrectly.  

---

## 7. Branch-and-Bound Pruning

Once we have partitions, the top-down enumeration recursively tries join orders. We can prune the search space using:

### 7.1 Accumulated-Cost Bounding
- Maintain a **global upper bound** U: the cost of the best physical plan found so far.  
- Maintain a **lower bound** L as we descend the tree, summing the cost of chosen physical operators along the path.  
- If `L + something > U`, prune.  
- Tends to prune more effectively (since it uses real operator costs) but can serialize search.

### 7.2 Predicted-Cost Bounding
- Maintain an upper bound per **logical expression** (reset to ∞ upon new expression).  
- Compute a **predicted** lower bound from logical properties (e.g., estimated cardinalities) before picking physical operators.  
- Avoid sub-expressions with high predicted cost.  
- Potentially more parallelizable, but might be less precise.

**Combined Approach**: Can apply both predicted and accumulated bounding. The paper’s experiments show:
- **Accumulated** cost bounding prunes more effectively overall, but can reduce concurrency.  
- **Predicted** cost bounding runs faster (less overhead in partial checks) but prunes less aggressively.  

---

## 8. Results & Observations

- Experiments on **synthetic query graphs** (SIGMOD 2007) measure **search complexity** rather than final plan cost.  
- **Min-cut** approach significantly reduces unnecessary enumerations compared to naive partitioning.  
- **Branch-and-bound** pruning strategies can yield large CPU-time reductions.  
- Accumulated bounding typically prunes deeper, predicted bounding runs faster.  

---

## 9. Extensions: Top-Down Hypergraphs

Real queries often contain:
- Complex multi-way join predicates (hyperedges).  
- Outer joins / non-inner joins.

**Next Steps**: Combine top-down min-cut partitioning with **hypergraph** representations (VLDB 2013 “Counter Strike”).  
- Convert hypergraphs to simpler graphs.  
- Apply min-cut logic.  
- Incorporate partial expansions as needed.

We will revisit this in the upcoming lecture on **search parallelization**, as top-down hypergraphs can further complicate concurrency.

---

## 10. Conclusion

- Top-down enumeration is intuitively appealing, with **branch-and-bound** and **demand-driven** transformations.  
- **OTDP** addresses cartesian product avoidance with a well-defined partitioning approach.  
- Pruning is crucial for **large** queries; different pruning strategies balance **quality vs. concurrency**.  
- In practice, building a real engine requires combining:
  - **Min-cut partitioning**  
  - **Memoization**  
  - **Adaptive bounding**  
  - Possibly **hypergraphs** for advanced predicates/outer joins