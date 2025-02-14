# CMU SCS 15-799 (Spring 2025): Optimizer Generators – Exodus and Volcano

## 1. Context & Motivation

### 1.1 Recap of Starburst
- Prior lecture covered **IBM Starburst**, a *stratified* optimizer that divides the optimization process into:
  1. **Rewrite Phase** (logical → logical, no cost model)
  2. **Plan Enumeration** (logical → physical, cost-based)
- Starburst introduced the idea that data **properties** (e.g., sort order, cardinalities) are first-class citizens during optimization.

### 1.2 Extensibility & Optimizer Generators
- Building a good query optimizer is extremely difficult.
- **Optimizer Generators** promise a framework (or toolkit) where DBMS implementers define:
  - Custom operators / transformations
  - Access methods
  - Basic rules
- The generator then produces an optimizer specialized to those definitions.  
- Ensures that the DBMS developer need not start from scratch every time.

---

## 2. Exodus Optimizer

### 2.1 Overview
- Developed as part of the **Wisconsin EXODUS** extensible DBMS project (late 1980s).
- **Rule-based approach** that separates:
  1. **Transformation rules** (logical → logical)  
  2. **Implementation rules** (logical → physical)
- Uses dynamic programming and **branch-and-bound** to prune search space.

### 2.2 First Implementation: Prolog
- Initially implemented transformations in **Prolog** (and LOOP), a logic programming language.
- Prolog’s built-in pattern matching and interpreter seemed appealing for rule-based systems.
- Ultimately **abandoned** due to:
  - **Fixed** depth-first search strategy in Prolog
  - Poor performance of their custom Prolog interpreter

Hence, the team moved to generating **C code** instead.

### 2.3 Compilation Pipeline

1. **Model Description File**:  
   - Specifies operator definitions, access methods, transformation + implementation rules, etc.
2. **Optimizer Generator**:  
   - Consumes the model file + user-supplied C operator implementations
   - Produces specialized C code
3. **Compilation & Linking**:  
   - Output compiled into a `.so` (shared object) and linked into the DBMS

At **query execution time**, the optimizer compiled by Exodus is invoked on the logical plan to produce a physical plan.

### 2.4 Search Algorithm & Data Structures
- Maintains two key data structures:
  - **MESH**: A hash table storing partial “access plans.”
  - **OPEN**: A priority queue of transformations to apply.
- Each iteration:
  1. Selects the best transformation rule from OPEN (based on an *expected cost factor*).
  2. Applies the transformation.
  3. Immediately applies implementation rules and updates cost calculations.
  4. Pushes new valid transformations onto OPEN.

#### Expected Cost Factor
- Each transformation rule has a factor *f* indicating how it scales the plan’s cost.
- If plan cost is *C*, after applying a rule with factor *f*, new cost is approximately *C × f*.
- They learn *f* dynamically from past transformations, adjusting factors for the last two rules used.

### 2.5 Observations & Limitations
1. **Coupled Logical & Physical**:  
   Each logical operator is stored together with its physical implementation(s) in MESH, leading to duplication.
2. **Property Enforcing**:  
   Logic for “satisfying physical properties” (e.g., sorting) is embedded in cost functions, rather than purely in rewriting rules.
3. **Immediate Implementation**:  
   As soon as a logical transformation is applied, Exodus tries to convert it to physical operators and do cost analysis—leading to a blow-up of possibilities.
4. **Learning Approach**:  
   Exodus attempts to prioritize beneficial rules (via the *expected cost factor*) but only within a single query optimization.

---

## 3. Volcano Optimizer

### 3.1 Project Background
- **Volcano** was a later project (early 1990s) by Goetz Graefe, building on lessons from Exodus.
- Also an **extensible** DBMS toolkit that:
  - Separates transformation logic from the search mechanism
  - Emphasizes a clean cost model and property framework
  - Implements a **top-down** (backward chaining) search strategy.

### 3.2 Design Goals
1. **Interoperable** with existing DBMSs
2. **Efficient** in computation & memory
3. **Extensible** physical properties
4. **Flexible** search pruning
5. **Supports** incomplete or parameterized queries (placeholder values)

### 3.3 Two Algebras: Logical & Physical
- **Logical expressions** represent what the query wants (e.g., a `JOIN` node in relational algebra).
- **Physical expressions** represent actual execution strategies (e.g., `HASH_JOIN`, `MERGE_JOIN`).
- Provides a consistent framework for transformations to move from logical to physical.

### 3.4 Enforcer Operators
- Special “virtual” physical operators that ensure certain data properties, like sorting or partitioning, are satisfied.
- Similar to Starburst’s “Glue” nodes, but integrated more systematically.

### 3.5 Rules & Cost Functions
- **Matching function** + optional *auxiliary function* (procedural checks).
- Physical operators have a **cost function** provided by the DBMS implementer.
- Logical operators have **no** cost; only the physical or enforcer operators do.

### 3.6 Top-Down Search with Memoization
- **Backward chaining** (depth-first):
  1. Start at the root: the final query result or sub-plan.
  2. Recursively expand transformations downward to find suitable logical or physical children.
  3. Use a **lookup table** (similar to a memo) to avoid re-analyzing the same sub-expression multiple times.
  4. Allows pruning of subplans once a better cost plan is found.
- **Key advantage** over Exodus:  
  Because Volcano memoizes sub-expressions, it avoids re-deriving or re-costing the same partial expressions.

### 3.7 Comparison with Exodus & Starburst
- **Performance**:  
  - Volcano is substantially faster than Exodus when queries involve >3 relations, as shown in the benchmark times on a Sun SPARCStation-1.
  - Avoids storing separate logical+physical pairs for each operator in the memo structure.
- **Single-Level Rules**:  
  - Volcano rejects Starburst’s approach of multi-layer expansions (QGM → STAR → LOLEPOP).  
  - Argues simpler single-level rules are easier to maintain and extend.
- **Still Expands All Equivalences**:  
  - Volcano enumerates all logically equivalent sub-expressions and possible physical implementations upfront (though uses top-down expansions).
  - Next-generation systems (like Cascades) improve further.

---

## 4. Summary & Key Takeaways

1. **Exodus & Volcano** are prototype “optimizer generators” that:
   - Use transformation & implementation rules to convert logical plans into physical plans.
   - Provide a rule compilation pipeline where code is generated from a high-level model description.
   - Attempt to unify or separate distinct concepts like cost, property enforcement, and transformation.

2. **Exodus**  
   - **Forward chaining** (bottom-up) approach.  
   - Immediate implementation of logical operators → physical, leading to large expansion.  
   - Introduced cost-based transformation prioritization with an *expected cost factor*.

3. **Volcano**  
   - **Backward chaining** (top-down) approach with a branching, memo-based search.  
   - Memoization helps avoid repeated re-analysis of the same subplan.  
   - Emphasizes using *single-level rules* plus *enforcer operators* to handle sorting/partition properties.  
   - Generally outperforms Exodus on queries with multiple relations.

4. **Comparison to Starburst**  
   - Volcano criticizes Starburst’s “hierarchical” grammar.  
   - Maintains that a single set of rules with straightforward pattern matching is more extensible.  
   - Both Starburst and Volcano introduced cost-based frameworks with first-class data properties, but Starburst organizes them differently (QGM, STAR, etc.).