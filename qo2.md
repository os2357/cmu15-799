# CMU SCS 15-799 | History of Query Optimizers and IBM System R

## 1. Background: Dawn of the Relational Model

### 1.1 Early Procedural DBMSs
- Late 1960s: Systems like CODASYL required developers to manually specify access paths and iteration flows for queries.
- Problem: As data or schemas changed, developers had to rewrite these procedural database programs to maintain performance.

### 1.2 Emergence of Declarative Queries
- Edgar F. Codd proposed the relational model (late 1960s / early 1970s).
- Key Insight: Provide a declarative language (SQL or relational algebra) so that:
  - The user states what data is needed (e.g. `WHERE album.name = 'Mooshoo Tribute'`).
  - The DBMS decides how to best retrieve it.

### 1.3 Early Relational Systems
- System R (IBM Research, San Jose)
- INGRES (UC Berkeley) -> led to PostgreSQL (i.e. "Post-INGRES")
- Oracle (Larry Ellison, commercializing ideas from IBM)
- Mimer (Uppsala University, Sweden)

All these systems had to address query optimization (deciding join order, access paths, etc.). Initially, most adopted heuristic-based approaches; System R uniquely pioneered a more systematic cost-based method in the 1970s.

---

## 2. Heuristic-Based Optimization

### 2.1 General Approach
A heuristic or rule-based optimizer:
1. Applies logical -> logical transformations (e.g., pushing down selections, converting Cartesian products to joins).
2. Chooses join order through simple rules, often ignoring detailed cost estimates.

**Examples**:
- INGRES in the 1970s
- Oracle until the early 1990s (famously used "the order in the FROM clause" for join ordering)
- Many new DBMSs start with heuristic optimizers due to simpler implementation

### 2.2 Relational Algebra Equivalences
The optimizer relies on standard equivalences:
- Selections can be split and pushed down.
- Joins can be reordered (commutative, associative).
- These rules allow safe logical transformations without a cost model.

### 2.3 INGRES Example
INGRES (1970s) used a heuristic approach that sometimes decomposed multi-relation queries into smaller, single-relation queries. For instance:

```sql
    SELECT ARTIST.NAME
      FROM ARTIST, APPEARS, ALBUM
     WHERE ARTIST.ID = APPEARS.ARTIST_ID
       AND APPEARS.ALBUM_ID = ALBUM.ID
       AND ALBUM.NAME = "Mooshoo Tribute"
     ORDER BY ARTIST.ID;
```

- Decompose into queries each accessing one table plus a temporary table.
- Substitute the results into subsequent queries, effectively skipping a comprehensive join optimization step.

This strategy worked for smaller or simpler data, but it was clearly suboptimal for more complex joins.

### 2.4 Pros and Cons of Heuristic-Only Approaches

- **Pros**:
  - Straightforward to implement (simple if-then-else rules).
  - Fast for basic queries.

- **Cons**:
  - Relies on hard-coded assumptions or "magic constants."
  - Fails for complex multi-way joins or correlated subqueries.
  - Often yields suboptimal plans.

---

## 3. Heuristics Plus Cost-Based Search: System R

### 3.1 Overview
IBM's System R project (1970s) introduced the first cost-based query optimizer. It still used heuristic rewrites for some transformations, but crucially introduced a dynamic programming search for join ordering. This method influenced almost all subsequent relational DBMSs.

**Key points**:
1. Perform initial logical-to-logical rewrites via simple heuristics (e.g., predicate pushdown).
2. For multi-relation query blocks, enumerate alternative join orders and choose the best one based on cost estimates.
3. Use statistics and a cost model to pick an optimal access path (e.g., index vs. sequential scan) for each table.

### 3.2 Single-Relation Queries
System R first focuses on single-table (single-block) queries. If the query is "sargable" (i.e., the predicates can use an index), the optimizer:
1. Considers each applicable index vs. a sequential scan.
2. Estimates cost using formulas based on:
   - Number of page fetches (I/O cost).
   - Weighted CPU usage (e.g., "RSI calls").
3. Selects the cheapest access path.

### 3.3 Cost Model and Selectivity
- **Cost** = (page fetches) + (weight * CPU operations).
- **Selectivity factor**: Fraction of tuples expected to match a predicate.
- The system uses simple assumptions (often uniform distribution) to estimate selectivity from stored table/column statistics.
- These assumptions can be inaccurate, but were pioneering at the time.

### 3.4 Interesting Orders
For queries that require sorted output (e.g., ORDER BY, GROUP BY), System R notes two possibilities:
1. An access path that inherently produces the desired order (e.g., an index scan matching the ORDER BY).
2. A cheaper-but-unsorted access path, plus an explicit sort at the end.

The optimizer compares both and picks whichever is cheaper overall.

### 3.5 Multi-Relation Queries
When multiple tables appear in the query block, the optimizer applies a dynamic programming approach to find the best join order:

1. **Start**: Pick the best single-relation plans (using step 3.2).
2. **Combine**: Enumerate pairs of relations, calculate join costs, and pick the cheapest.
3. **Iterate**: Combine partially joined subsets with remaining tables, building up to the final join.

This classic "bottom-up" or "generative" search restricts itself to "left-deep" plans by default to manage complexity (avoiding a blow-up in enumeration).

### 3.6 Nested Queries
- System R treats nested subqueries as separate query blocks.
- The optimizer fully plans and executes the inner query, then substitutes its result (or materializes it to a temp table) in the outer query.
- Modern systems often rewrite such subqueries into joins or correlated evaluation, but System R's approach was a major step forward in the 1970s.

### 3.7 Advantages and Disadvantages

- **Advantages**:
  - Substantially better plans than naive heuristic-only approaches.
  - Formalized the concept of a cost model plus dynamic programming for join order selection.

- **Disadvantages**:
  - Relies on simplifying assumptions about data distributions (uniformity, independence).
  - Only considers left-deep plans, which are not always optimal.
  - Sorting or other physical properties are treated in an ad-hoc way (i.e., "interesting orders" logic rather than integrated into a unified search).

---

## 4. Takeaways

1. **Heuristic vs. Cost-Based**:
   - Heuristics alone are insufficient for complex queries.
   - System R introduced a dynamic programming approach that remains a foundation for modern DBMS optimizers.

2. **Bottom-Up Enumeration**:
   - System R enumerates plans from single-relation to multi-relation via incremental combination.
   - Although 40+ years old, this technique is still used in many systems today.