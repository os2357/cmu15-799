# CMU SCS 15-799 (Spring 2025): IBM Starburst Query Rewriter & Optimizer

## 1. Background & Motivation

### 1.1 System R Limitations
- **System R** selected each table’s access method before enumerating join orders, rather than choosing them **in tandem**.
- By the mid-1980s, new database workloads arose (object-oriented databases, active/triggers) that required **extensibility** in both the runtime engine and the optimizer.

### 1.2 The Rise of Extensible DBMS
- **IBM Starburst** (starting 1985) aimed to let developers easily add new access methods, data types, and operators—essentially bridging the gap between purely relational systems and emerging “object-relational” needs.
- A key insight: **adding new functionality** requires the query optimizer to be aware of it. Hence, a new, more flexible optimization framework.

---

## 2. Optimizer Generators & Stratified Search

### 2.1 Optimizer Generators
- Tools or frameworks that **separate** the optimization search strategy from the transformation rules/operators themselves.
- Goal: Let DBMS developers write rules to define *how* to optimize queries (rule-based rewrite, cost-based planning) while reusing a generic search engine.

### 2.2 Stratified vs. Unified Search
- **Stratified Search**:  
  1. Perform **rewrite** transformations (heuristics) on the query’s logical form (no cost-based decisions).  
  2. Then, do **cost-based** physical plan enumeration.  
  - Examples: **Starburst**, CockroachDB.
- **Unified Search**:  
  - Combine all transformations—logical and physical—into one memoized search.  
  - Examples: **Volcano/Cascades**, Microsoft SQL Server, Orca.

In this lecture, we focus on **IBM Starburst** and its stratified pipeline.

---

## 3. IBM Starburst Overview

### 3.1 IBM DB History (1970s–1980s)
- **System R** (1970s): First cost-based optimizer, dynamic programming approach.
- **R*** (late 1970s): Distributed version of System R.
- **DB2** (1980s): Commercial product with partial System R technology.
- **Starburst** (mid-1980s): Research project to build an *extensible* DBMS at IBM Almaden, eventually influencing DB2 for LUW (Linux/Unix/Windows).

### 3.2 Architecture
Starburst’s **optimizer pipeline** separates query processing into:
1. **Parser + Binder**: Validates SQL, produces an initial logical representation.
2. **Query Graph Model (QGM)**: A higher-level, calculus-based representation.
3. **Rewrite Stage**: Applies rule-based, cost-agnostic transformations on the QGM.
4. **Plan Optimization**: Converts the rewritten QGM into a physical plan via cost-based search.
5. **Plan Refinement**: Final adjustments to produce an executable plan.

---

## 4. Relational Calculus & the Query Graph Model

### 4.1 Why Relational Calculus?
- **Relational algebra** is procedural: it specifies *how* the query executes.
- **Relational calculus** is nonprocedural: it specifies *what* the query wants, without ordering steps.
- Starburst uses a **Query Graph Model (QGM)**, inspired by tuple relational calculus, to represent queries at a higher level.

### 4.2 QGM Basics
- A QGM has **“boxes”** for `SELECT` statements (and nested subqueries).
- Each box has:
  - A **head** (output schema + distinct properties),
  - A **body** (iterators/quantifiers that read from either stored tables or other boxes),
  - **qualifiers** (predicates).
- Boxes connect to stored tables or other boxes, forming a graph-like representation.
- Distinctness properties track whether outputs must enforce uniqueness (`ENFORCE`, `PERMIT`, etc.).

### 4.3 Example
Consider:
```sql
SELECT DISTINCT q1.partno, q1.descr, q2.suppno
  FROM inventory AS q1, quotations AS q2
 WHERE q1.partno = q2.partno
   AND q1.descr = 'engine'
   AND q1.price <= ALL(
     SELECT q3.price
       FROM quotations AS q3
      WHERE q2.partno = q3.partno
   );
```

In Starburst’s **Query Graph Model (QGM)** representation, each nested subquery initially appears as a separate `SELECT` box. The **Rewriter** merges these boxes into a single box (when valid) to expose more opportunities for the cost-based optimizer to reorder joins and choose efficient access methods. By rewriting nested subqueries into equivalent joins, Starburst’s rewriter converts “procedural-like” queries into declarative, single-box QGMs.

---

## 5. Plan Optimization with LOLEPOPs and STAR Rules

After rewriting, Starburst transforms the QGM into a physical plan. This second optimization phase is **cost-based** and guided by **STAR** (Strategy Alternative Rules). These rules specify how high-level operations (non-terminals) become low-level physical operators (terminals), called **LOLEPOPs** (“LOw-LEvel Plan OPerators”).

### 5.1 LOLEPOPs
- LOLEPOPs are Starburst’s physical execution operators (e.g., table `ACCESS`, `JOIN`, `SORT`).
- Each LOLEPOP always produces a single output “table” (or stream) from one or more inputs.
- **Parameters** (or “flavors”) indicate which algorithm is used (e.g., nested-loop vs. hash join).

**Examples** of LOLEPOPs:
- `ACCESS`: read data from a stored table or index.
- `JOIN`: merges two streams/tables; could be parameterized as `Hash`, `Sort-Merge`, or `Nested-Loop`.
- `SORT`: sorts the incoming data stream on specified columns.
- `STORE`: writes a stream to a physical table (often used for temp materialization).

### 5.2 STAR Rules
- **STAR** = “STrategy Alternative Rules.”
- They define how to generate physical operators (LOLEPOPs) from higher-level constructs in the QGM.
- Each rule can have multiple alternatives, effectively creating different physical plan branches. For instance, a `JOIN` could transform into:
  1. `NLJOIN` if indexing or small outer table is expected,
  2. `HSJOIN` (hash join),
  3. `MGJN` (merge join), etc.

These expansions produce multiple plan alternatives, each with estimated cost. The **cost model** considers I/O, CPU, and additional factors (sort orders, indexes, etc.), and the optimizer picks the cheapest plan.

### 5.3 GLUE and Plan Properties
- Starburst tracks *plan properties* (e.g., required sort order, distinctness, cost estimates) as first-class citizens in the plan.
- **GLUE**: a special STAR that ensures the final plan satisfies required properties. For instance, if the plan’s output must be sorted, GLUE might insert a `SORT` LOLEPOP if no preceding operator provides that property.

---

## 6. Search Termination

Like most cost-based optimizers, Starburst can’t explore all possible plans if the search space is huge. It employs **early stopping** mechanisms:
1. **Wall-clock Time**: stop after a configured maximum optimization time.
2. **Cost Threshold**: stop if a plan is found below a certain cost.
3. **Exhaustion**: once no new transformations are found (common for small sub-plans).
4. **Transformation Count**: stop after applying a certain number of rules.

In practice, DB2 LUW (the commercial successor of Starburst) uses a combination of thresholds and heuristics to limit optimization time.

---

## 7. Summary & Key Takeaways

1. **Stratified Approach**:  
   - Starburst divides query optimization into two stages:
     1. **Rewrite** (QGM-based transformations, no cost considerations)  
     2. **Cost-based planning** (using STAR rules + LOLEPOPs)

2. **Higher-Level Representation**:  
   - Starburst introduced a more advanced internal model (QGM) based on **relational calculus**, not relational algebra.  
   - This makes it easier to un-nest subqueries and unify them in a single plan representation.

3. **Extensibility**:  
   - Starburst was designed to accommodate new data types, user-defined functions, and access methods.  
   - The optimizer’s rules can be extended to handle these new “operators.”

4. **Influence on DB2**:  
   - Techniques prototyped in Starburst were integrated into IBM’s DB2 (Linux/Unix/Windows) product line.  
   - Showed that rewriting + cost-based search is a robust strategy for extensible, industrial-strength optimizers.