# CMU SCS 15-799 | Parallelization: Bottom-Up

## 1. Recap from Lecture 8

We explored **top-down** join enumeration and how partitioning the query graph into subgraphs can guide and prune the search space more effectively. However, different bounding strategies (e.g., accumulated-cost vs. predicted-cost) introduced data dependencies that could prevent easy parallelization.

---

## 2. Parallel Query Optimization: Motivation

A **parallel** optimizer attempts to generate better plans **faster** by leveraging multiple workers (threads/cores/GPU). Ideally:
- **Search time** scales **linearly** with the number of workers.
- All workers remain busy (no idle time).
- Data dependencies and synchronization overhead should be minimal.

**Why is this hard?**  
- The optimizer typically competes for resources with query execution.  
- Modern DBMSs usually run a single-threaded optimizer, letting multiple queries run in parallel.  
- **Non-serial, polyadic** DP algorithms (like those used for join enumeration) complicate concurrency due to deeper data dependencies.

---

## 3. Overview: DP Algorithms for Join Enumeration

### Non-Serial, Polyadic DP
- **Non-serial**: The computation in one phase can depend on results from multiple previous phases (not strictly the immediately preceding one).  
- **Polyadic**: Each recurrence step involves more than one subproblem/term (e.g., left and right subexpressions in a join).

The challenge is splitting up these subproblems so multiple workers can progress independently.

---

## 4. DPSub (Dynamic Programming Subset)

Reference:  
“**Efficient Massively Parallel Join Optimization for Large Queries**,” SIGMOD 2022

### 4.1 High-Level Idea
- Enumerate **all subsets** of relations in order of increasing size.
- For each subset **S**, consider all valid ways to split it into `S_left` and `S_right`, ensuring no disconnected subgraphs (i.e., valid join predicates exist).
- Calculate the **best plan** for **S** by combining the plans for `S_left` and `S_right`.

This is analogous to the traditional **System R** DP approach, except it explicitly enumerates subsets in increasing size. In parallel:
- We can distribute different subset checks across multiple workers.

### 4.2 Example
1. **Subsets of size 2**: Evaluate all join combos (A-B, A-C, B-C, etc.).  
2. **Subsets of size 3**: Combine smaller subsets.  
3. … continue until the full set.

---

## 5. MPDP (Massively Parallel DP)

Reference:  
“**Efficient Massively Parallel Join Optimization for Large Queries**,” SIGMOD 2022

### 5.1 Adaptive Approach
- For **simple** queries: run a parallelized version of DPSub.  
- For **larger** queries: combine **heuristics** (like Goo or cost-based grouping) with DP to prune the massive search space.

### 5.2 Vertex vs. Edge Enumeration
- **Vertex-based** (DPSize / DPSub): enumerates subsets by relation vertices. Simpler but can evaluate many **invalid** pairs.  
- **Edge-based** (DPHYP, DPCCP): enumerates possible joins in line with the graph’s edges. Fewer invalid pairs, but more complicated to parallelize.

### 5.3 UnionDP
- Split the query graph into **subgraphs** of size **k** (picked based on estimated cardinalities).  
- Use MPDP (the parallel DP technique) on each subgraph.  
- Combine partial solutions for a final plan.

### 5.4 GPU Implementation
- Represent the join graph with **fixed-width bitmaps**.  
- **Remove conditionals** to reduce **branch divergence** among GPU threads.  
- Send all cost estimates (e.g., cardinality stats) to GPU so it does not stall waiting on CPU calls.

**Empirical Results**:
- On some queries, the GPU approach shows significant speedup compared to single-thread CPU.  
- Plan quality is generally competitive, especially for large queries that produce many possible enumerations.

---

## 6. MPJ (Multiple Plan Join)

Reference:  
“**Parallelizing Query Optimization**,” VLDB 2008 (IBM Research)

### 6.1 Main Idea
- Store intermediate plans in a **memo table**, represented as a relation in the DBMS (self-joins on this relation enumerates new plans).  
- Each tuple includes:
  - **Quantifier Set**: which relations (or subqueries) are included.
  - **Plan List**: possible ways to produce those quantifiers.
- **Partition the search space** into subproblems by **quantifier-set size**.

### 6.2 Self-Join Example
1. **Partition P1**: size=1 (leaf relations).  
2. **Partition P2**: size=2. Combine the subplans from P1 with themselves.  
3. **Partition P3**: size=3. Combine P1 and P2, etc.  

Assign these partitions to different workers so they can proceed in parallel. The key is to avoid a big “cartesian product” of plan combos where many are invalid.

---

## 7. Allocating the Search Space

### 7.1 Total Sum Allocation
- Compute the total number of potential plan joins to explore and **split evenly** among workers.  
- Easiest but can lead to **imbalance** (some workers discover invalid subplans and become idle).

### 7.2 Stratified Allocation
- **Divide** the search space based on the loops in the DP enumeration logic (outer loop for subset sizes, inner loop for combining partitions).  
- **Equi-Depth**: break the outer loop into contiguous ranges.  
- **Round-Robin Outer**: loop iterations distributed to workers in a round-robin fashion.  
- **Round-Robin Inner**: each iteration’s inner loop tasks distributed in round-robin.

**Goal**: keep every worker busy and avoid wasted time on invalid subplans.

---

## 8. Parting Thoughts

- **Dividing tasks for parallelizing a DP-based join search** is **non-trivial**.  
- Using a **dedicated GPU** purely for query optimization is unlikely to be cost-effective in production settings. However, integrated GPUs (APUs) might be a future path if they can accelerate the search cheaply.  
- Many modern DBMSs still use single-threaded optimizers due to complexity and resource trade-offs.