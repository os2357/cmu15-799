# CMU SCS 15-799 (Spring 2025): Join Ordering: Bottom-Up

## 1. Introduction

This lecture focuses on the challenge of **join order enumeration** in query optimization. Building on Lecture 6’s discussion of query transformations, we now explore systematic approaches to **generate efficient join orders** for queries, especially as query complexity grows.

The key insight from the paper discussed is that **different query complexities require different enumeration strategies**:

- **Small Queries**: Exhaustive search (Dynamic Programming) is feasible.
- **Medium Queries**: Linearization techniques help simplify enumeration.
- **Large Queries**: Greedy and randomized approaches can approximate good plans.

---

## 2. Key Observations

- Most real-world queries involve **only 1-2 tables**.
- Complex queries joining **hundreds or thousands of tables exist** but are typically **machine-generated** (e.g., SAP’s 5000-table join).
- **Join ordering is NP-hard**, so exhaustive enumeration becomes impractical beyond a certain number of tables.

---

## 3. Complexity Classes of Join Queries

A query’s complexity is driven by **its join graph structure**:

1. **Chain Query**: Each table joins to at most two others.
2. **Clique Query**: Every table joins with every other table.
3. **Star Schema**: Fact table joins with many dimension tables.

Real-world queries often **fall between chain and clique structures**.

---

## 4. Join Enumeration Strategies

### **4.1 Small Queries: Dynamic Programming with Hypergraphs**
- For **10 or fewer relations**, **exhaustive enumeration** is feasible.
- Uses **Dynamic Programming (DP)** with **Hypergraphs** for efficiency.
- **Hypergraph Representation**:
  - Groups relations into **subgraphs** with **hyperedges**.
  - Treats **subgraphs as units** to **prune redundant search paths**.
- **Optimal Plan Guaranteed**.

#### **Key Benefit:**
- **Avoids redundant exploration** by grouping subproblems.

---

### **4.2 Medium Queries: Search Space Linearization**
- For **11 to ~100 relations**, full enumeration is impractical.
- **IKKBZ Linearization Algorithm (1980s)**:
  - Converts **join graph into a linear chain**.
  - Produces a **left-deep tree approximation** in **O(n²)** time.
  - **Ranks relations by selectivity & cost** (cost-benefit ratio).
  - Merges based on **precedence and rank**, **collapsing subtrees** if necessary.

#### **Key Benefit:**
- **Fast heuristic approximation**.
- Serves as **a good starting point** for further refinement with **DP-based bushy plan search**.

---

### **4.3 Large Queries: Greedy & Adaptive Refinement**
- For **100+ relations**, exhaustive search is infeasible.
- **Greedy Operator Ordering (GOO)** algorithm:
  - Iteratively picks **the cheapest join (size × selectivity)**.
  - Merges subplans into **a single node**.
- **Adaptive Subplan Refinement**:
  - Identifies **expensive subplans** (size K, e.g., K=4 to 7).
  - **Optimizes subplans** locally using **DP**.
  - Repeats until **time budget is exhausted**.

#### **Key Benefit:**
- **Graceful degradation** as complexity increases.
- Balances **plan quality** and **optimization time**.

---

## 5. Randomized Algorithms (Rarely Used)

PostgreSQL’s **Genetic Query Optimizer (GEQO)** and other **randomized approaches** were discussed but are **generally discouraged**:

### **5.1 QuickPick**
- **Randomly samples edges**, builds join trees.
- Keeps **best plan seen so far**.

### **5.2 Simulated Annealing**
- **Randomly mutates plans**.
- Accepts **worse plans probabilistically** to **escape local minima**.

### **5.3 PostgreSQL’s Genetic Algorithm (GEQO)**:
- **Enabled for queries with ≥13 tables**.
- **Crossbreeds subplans**, selects **best offspring** iteratively.
- **Deterministic seed** ensures **consistent plans**.

#### **Key Issues:**
- **Opaque, non-explainable behavior**.
- **Fragile and outdated**.

---

## 6. Performance Results (Paper Evaluation)

- **Umbra’s Adaptive Approach (DP + Linearization + GOO)**:
  - Handles **hundreds of relations in < 500 ms**.
  - Produces **plans within 10% of optimal cost**.
- **Commercial Systems** (Oracle, SQL Server):
  - **Struggle beyond ~50-300 tables**.
- **PostgreSQL (GEQO)**:
  - **Breaks down after ~700 tables**.

---

## 7. Final Takeaways

- **Join order optimization requires hybrid strategies**.
- **Adaptive approaches** based on **query complexity** (Small, Medium, Large) **yield the best results**.
- **Dynamic Programming with Hypergraphs** is **state-of-the-art** for **small and medium queries**.
- **Linearization (IKKBZ)** and **Greedy Refinement** **help large queries gracefully degrade**.