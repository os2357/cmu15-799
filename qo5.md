# CMU SCS 15-799 (Spring 2025): Cascades Optimizer

## Overview

This lecture delves into **Cascades**, the third optimizer framework developed by Goetz Graefe after Exodus and Volcano. Compared to the earlier optimizers, **Cascades** unifies logical→logical and logical→physical transformations, but does so *lazily* and on-demand. It follows a **top-down, goal-oriented search** similar to Volcano (“backward chaining”) but introduces several improvements:

1. **Fine-Grained Tasks** – Instead of a purely recursive (call-stack) approach, each “chunk of work” is treated as a task.  
2. **Flexible Scheduling / Promises** – The system can reorder or prioritize tasks based on heuristics, properties, or other knowledge.  
3. **Unified Representation** – Both logical and physical rules run through the same “search engine,” with more direct ways to handle properties (e.g., sort orders) and transformations.  
4. **Rules for Enforcers** – Instead of having dedicated “virtual operators” for guaranteeing properties (like sort order), Cascades models them as rules.  

Cascades was never released as a single reference implementation. Instead, it became the basis (or inspiration) for several commercial systems (e.g., **Microsoft SQL Server**), as well as open-source or academic systems like **Orca** (Greenplum) and **CockroachDB**’s newer optimizer.

---

## Volcano Recap & Cascades Motivation

### Volcano Limitations
- Volcano also used **top-down** search with a memo structure. It enumerated all logical expressions early (the “Generation” phase) and then converted them into physical expressions in a later phase.  
- Physical properties (e.g., sorting) were handled via “enforcer operators,” which had to be explicitly inserted into the plan tree.  
- The system would eagerly expand *all* logical expressions before doing cost analysis on them, potentially blowing up the search space.  

### Cascades Goals
- **Lazy expansion:** Only explore or rewrite sub-expressions on demand, according to task scheduling.  
- **Unify** all transformations (logical → logical, logical → physical, plus property enforcers) in a single search engine.  
- **Parallel-friendly** – By replacing the naive recursion with a “task queue,” it becomes easier to (in principle) share optimization work and run multi-threaded.  
- **Promises** – Provide flexible heuristics that can reorder or skip certain transformations if some rule is deemed more beneficial to explore sooner.

---

## Main Ideas in Cascades

### 1. Expressions and Groups

- An **expression** is a node/operator (logical or physical) with zero or more child expressions.  
- Multiple expressions that produce the same logical output are grouped together in a **Group**.  

Groups also capture the notion of equivalences: 
- **Logical** equivalences: e.g., `Join(A,B)` vs. `Join(B,A)` or re-associated multi-joins.  
- **Physical** equivalences: e.g., a hash join vs. a merge join producing the same final schema.

A **Group** is thus a container with:
- Some “logical signature” or “logical output” that the expressions produce.  
- A set of alternative logical expressions.  
- A set of alternative physical expressions.  
- Possibly the best (lowest cost) physical expression chosen so far, along with the cost.  
Groups reduce duplication in the memo table by letting multiple references point to a single Group that has already been expanded.

### 2. Memo Table

- Cascades keeps a global **memo** data structure that records each Group and all expressions (logical or physical) known to be in that Group.  
- Before expanding an expression or sub-expression, Cascades checks whether it already exists in the memo. If so, it can reuse prior expansions and cost computations.  
- Because “equivalent sub-queries” unify in the same Group, it avoids re-doing transformations for repeated sub-expressions.

### 3. Rules (Transformations & Implementations)

- **Transformation Rules (Logical → Logical):** e.g., commutativity, associativity, projection push-down, etc.  
- **Implementation Rules (Logical → Physical):** e.g., `LogicalJoin → HashJoin` vs. `LogicalJoin → MergeJoin`.  
- **Enforcer “Rules”:** In Volcano, an “enforcer operator” (like sorting) had to be inserted as a plan node. In Cascades, you can handle these as a type of transformation that states “If you need sort-order property on a child, apply a rule to get a Sort-PhysicalExpr.”  

### 4. Tasks and the Search Engine

Cascades moves away from a purely recursive search. Instead:

1. **Task** = a small piece of optimization work: “Explore this group,” “Implement this expression,” “Optimize child group,” etc.  
2. All tasks go onto a global **task queue** (often implemented as a stack). The optimizer picks tasks off the queue.  
3. **Promises** – Each rule or sub-task can be assigned a promise or priority that indicates how beneficial we expect it to be. This helps the system reorder tasks instead of always using a strict LIFO or DFS approach.  

Hence, the search can jump around the memo structure, focusing first on transformations with higher promise, or diving into children if needed, rather than systematically enumerating everything.

---

## Example Search Flow (High Level)

1. **Start** with a single Group for the entire query at the root: no expressions, just the “logical signature” of the final output.  
2. Insert a “task” to explore that Group’s logical expressions. Possibly:
   - Add a new logical expression: e.g., `Join(A,B,C)` → `[Join( Join(A,B), C ), ...]`.  
   - Then schedule tasks to expand sub-groups for `(A,B)` or `(A,B)⨝C`.  
3. Eventually, once you add an Implementation Rule (logical → physical), a new physical expression appears in a Group.  
   - The system calculates cost using the child groups’ best known costs.  
   - If that total cost is lower than the best known cost for that Group, you record it.  
4. Once the root Group obtains at least one valid physical plan, you have a feasible plan. The system can keep exploring if it thinks it can find cheaper alternatives, or if the time/budget allows.  

---

## Promises

- A rule or transformation can declare a “promise” (an integer, real number, or some ranking) indicating how beneficial or urgent it might be.  
- During the top-down search, the system can pick tasks with higher promise first.  
- This approach generalizes or replaces the approach from Exodus that embedded cost directly in “expected cost factors.”  
- Because we may not have a cost (no physical operator chosen yet), a promise can also be based on heuristics (e.g., “pushing down filters is always a top priority,” “commutativity might help if a child is significantly smaller,” etc.).

---

## Implementation Variants & Real Systems

### 1. Microsoft SQL Server

- Began adopting Cascades in ~1995. Entirely in C++.  
- Instead of a pure “unified search,” it internally performs multi-phase optimization:
  1. **Pre-Optimization:** Basic rewrites (constant folding, simplification, removing trivially-empty subqueries).  
  2. **Phase 0**: Quick transformations, cheap queries might finalize here.  
  3. **Phase 1**: More advanced transformations, includes parallel or distributed.  
  4. **Phase N**: Possibly extra transformations for big queries.  
- A single codebase can handle multiple “forks” of SQL Server (e.g., big data vs. small OLTP).  
- Distinguishes each stage by enabling certain rules (and associated promises) or by having a threshold on how many transformations can be attempted.

### 2. Orca (Greenplum / Pivotal)

- Orca is a standalone C++ library for query optimization, based on Cascades.  
- By design, multiple DBMS front-ends (Greenplum, HAWQ, etc.) can call Orca to produce a plan.  
- Implements parallel search (multi-threaded) by using a shared memo and concurrency control.  
- Was open-sourced around 2013, but as of ~2024, active development is less visible due to corporate acquisitions.

### 3. CockroachDB

- CockroachDB’s newer optimizer (post-2018) is a Cascades-inspired design, also single-threaded.  
- Uses a DSL (Optgen) to define rules, which get compiled to Go code.  
- Extends the approach to handle partial indexes, multi-tenant constraints, etc.

### 4. Others

- Clustrix (renamed “MariaDB Xpand”), Snowflake, Databricks – each has something similar, but details are not always public.  
- CMU’s NoisePage and OtterTune-based prototypes also consider using a Cascades skeleton for experimentation.  

---

## Pros & Cons of Cascades

**Pros**
1. **On-demand** transformation search: Only expand sub-expressions when (and if) needed.  
2. **Expressive**: A single set of rules can represent rewrites (logical→logical) or implementations (logical→physical).  
3. **Properties as Rules**: Sorting, partitioning, or distribution can be handled in a uniform manner.  
4. **Memo** with “small tasks” → easier to prune or reorder expansions than in naive bottom-up enumerations.  
5. Encourages engineering discipline: Different teams can maintain separate sets of rules for various features.

**Cons**
1. The system can become complex. Tuning “promises,” or multi-phase expansions, can be tricky.  
2. Code generation for new operators (and rule debugging) can be non-trivial in practice.  
3. Strictly top-down might still examine more partial plans than a well-tuned dynamic programming approach for multi-joins (the “Hyper” style).  
4. Real commercial systems (SQL Server, etc.) add numerous heuristics and “macro-rules” that reduce the theoretical cleanliness in favor of real performance.

---

## Wrap-Up

Cascades is widely regarded as **the** most influential next-step after Volcano for query optimization frameworks. Although the original paper never shipped a reference codebase, many real systems (SQL Server, Greenplum Orca, CockroachDB) show that a Cascades-like approach can remain maintainable and produce high-quality plans at scale.

**Key differences from Volcano**:

- **No forced 2-phase** expansion: Instead, transformations happen on demand through tasks.  
- **Promises** let the system pick “better rules” earlier.  
- **Properties** and enforcers become simply more rules in the same engine.  
- The **task queue** approach is more flexible than pure DFS recursion.  

In commercial practice, large queries often run in multi-phase or multi-stage mode (like in SQL Server), gating which transformations become legal at each stage, giving an almost “stratified” flavor on top of a Cascade foundation.