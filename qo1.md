# CMU SCS 15-799 (Spring 2025): Introduction to Query Optimization

---

## 1. Course Overview

### 1.1 Course Objectives
- **Primary Goal**: Learn about modern query optimization practices and system-level programming techniques used in database query optimizers.
- **Focus Areas**:
  - Internal architecture and implementation of query optimizers
  - Writing correct and performant system code for databases
  - Proper documentation and testing
- **Scope**: 
  - Foundational materials (historical and core techniques)
  - State-of-the-art research and modern, real-world implementations

### 1.2 Why This Course?
- **Databases are ubiquitous**: There are more databases than ever, supporting diverse workloads.
- **Humans are not scalable**: Automated query optimization is crucial for efficient data processing as data size and system complexity grow.
- **Query optimization differentiates systems**: Many modern systems share similar execution architectures; the quality of the query optimizer often determines whether one system outperforms another.
- **Industry need**: Skilled engineers who can work on query optimization are in demand; many companies pay highly for experts in this area.
- **Research potential**: Query optimization remains an unsolved and NP-complete problem in its general form.

### 1.3 Background & Assumptions
- You should have completed an **introductory database systems course**
  - Assumes knowledge of SQL, relational algebra, query processing models, and general DBMS architecture.
- **Not** a machine learning course:  
  - Some optimization techniques may involve ML (e.g., cost estimation), but detailed ML methods will not be the focus.

---

## 2. Query Optimization: Background & Motivation

### 2.1 Why We Need Query Optimization
- **Declarative Language (SQL)**: 
  - Applications only specify *what* data they want, not *how* to retrieve it.
- **Optimizer’s Role**: 
  - Convert the user’s high-level request into a *physical* execution plan that runs efficiently and correctly.

### 2.2 Example: Drastic Performance Differences
Professor Pavlo provided a running example where two tables (`Emp` and `Dept`) are joined to retrieve distinct employee names from the “Toy” department. Several possible plans yield **huge performance variance**:
1. **Naive Cartesian Product** → ~2 million I/Os
2. **Join with basic page nested-loop** → ~54k I/Os
3. **Sort-merge join** with materialization → ~7k I/Os
4. **Sort-merge or hash join** with vectorized pipelining → ~3k I/Os
5. **Predicate Pushdown + Index Nested-Loop** → ~37 I/Os  

The difference between 2 million I/Os and 37 I/Os highlights *why* a good optimizer can speed up queries by **orders of magnitude**.

### 2.3 Typical DBMS Architecture (Parsing → Optimization → Execution)
1. **Parser**  
   - Converts SQL string to an Abstract Syntax Tree (AST).
2. **Binder / Resolver**  
   - Maps AST tokens (table/column names) to internal catalog identifiers (object IDs, schema info).
3. **Logical Plan**  
   - High-level relational algebra representation (e.g., `Scan A`, `Join B`, `Filter C`) without specifying physical details (e.g., which join algorithm to use).
4. **Optimizer**  
   - Searches a large space of logically equivalent plans.
   - Uses a **cost model** plus statistics (from the system catalog) to pick the best physical plan.
5. **Physical Plan**  
   - Complete blueprint specifying physical operators (e.g., hash join vs. sort-merge join vs. nested-loop).

---

## 3. Core Concepts in Query Optimization

### 3.1 Logical vs. Physical Plans
- **Logical Operators**: Abstract data flow (e.g., *join*, *filter*, *aggregate*) without prescribing *how* to execute.
- **Physical Operators**: Define specific algorithms or access paths (e.g., *hash join*, *index nested-loop join*, *sort-based aggregation*).
- **Not 1:1**: A single logical operator can map to multiple possible physical operators or even multiple combined operators.

### 3.2 Search Strategies

1. **Heuristic/Rule-Based**  
   - Apply general transformations (e.g., *always push down predicates*) that are known to improve performance in most cases.
   - Often used in early DBMSs or “first-pass” rewrite steps in modern systems.

2. **Cost-Based**  
   - Exhaustively (or strategically) explore many possible plans.
   - Compute/estimate a “cost” metric (I/Os, CPU usage, or a composite) for each plan.
   - **Select plan** with the lowest predicted cost.
   - Relies heavily on an accurate **cost model** and **statistics** (e.g., table sizes, data distribution).

### 3.3 Top-Down vs. Bottom-Up Optimizers
- **Bottom-Up** (Dynamic Programming Style):
  - Start with base relations (tables), build optimal plans for small sub-problems, then combine them to handle larger sub-expressions.
  - Common in System R, Starburst, Hyper, and many classical approaches.
- **Top-Down** (Transformational / Cascades Style):
  - Start with the full logical query and progressively refine or transform sub-expressions until arriving at a physical plan.
  - E.g., Volcano, Cascades, and (in practice) Microsoft SQL Server’s optimizer.  

(Some modern optimizers blend both.)

### 3.4 Objective Functions & Cost Models
- **Common Goals**:
  - **Minimize response time** or **resource consumption** (e.g., CPU, I/O)
  - **Maximize throughput**
  - **Minimize monetary cost** (particularly relevant in cloud usage or “serverless” contexts)
- **Single-Query vs. Multi-Query**:
  - *Single-Query* is standard in most systems; focuses on finding one good plan.
  - *Multi-Query Optimization* tries to find globally optimal strategies across a *batch* of queries (rare in production, more common in research).
- **Modeling Approach**:
  - The **cost model** is system-internal. Its numeric estimates are only used to compare candidate plans for *that DBMS*.
  - The DBMS rarely (if ever) runs all plan variants physically (an obvious time cost). It uses formulas and statistics to approximate.

### 3.5 Complexity & Challenges
- Query optimization is **NP-complete** in its full generality (especially for complex SQL features).
- “Optimizing” does not guarantee a mathematically optimal plan. Instead, DBMSs rely on:
  - **Heuristics**
  - **Dynamic programming** with pruned search
  - **Approximate or partial cost models**
  - Potentially **adaptive** query execution that adjusts on the fly.

---

## 4. Conclusion & Next Steps

### 4.1 Key Takeaways
- **Query Optimization** is central to DBMS performance.  
- Small mistakes in join orders or pushdown strategy can cause *massive* performance hits (e.g., 2M I/Os vs. 37 I/Os).
- Even the best optimizers do *not* guarantee truly optimal plans—only “good enough” given limited time and approximate statistics.