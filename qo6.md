# CMU SCS 15-799 | Transformations

## 1. Introduction

This lecture focuses on *query plan transformations* in a DBMS's optimizer. We continue from our previous understanding of how a Cascades-like system (or any top-down / bottom-up approach) is structured. Now, we look into the particular transformations that the optimizer uses to rewrite query plans. These transformations ensure semantic equivalence — the final query plan produces the same correct result as the original.

---

## 2. Recap & Motivation

Last lecture, we discussed:

- **Cascades** optimizer design (task-based transformation scheduling, deferred expansion, promise-based guidance, memo structures).
- The three main contributors to plan quality:
  1. **Search Algorithm** (top-down vs. bottom-up)
  2. **Cost Model** (statistics, cardinality, CPU & I/O cost, etc.)
  3. **Transformations** (rewriting the query plan to unlock new shapes)

We are now focusing on the *Transformations* aspect. The better the transformations, the more likely we can produce an optimal (or near-optimal) physical plan.

---

## 3. Transformations Overview

A **transformation** changes a query plan to a new form that is logically correct (i.e., produces the same output) but may have different performance characteristics. The motivation is:

- Potentially lower execution cost (push down predicates, reordering joins, etc.).
- Enabling additional transformations if needed (e.g., outer-join to inner-join conversion unlocks normal join reorderings).

In many systems, transformations are expressed as pattern matching rules. For instance:

```
IF (Filter --> Join) AND (Predicate references only columns in one side)
THEN pushdown the Filter
```

We also rely on relational algebra equivalences and domain knowledge (e.g., schema constraints, foreign keys, indexes, etc.).

---

## 4. Access Path Transformations

A *base relation* (a table in the FROM clause) can be accessed via multiple methods:

- **Table Scan** (sequential)
- **Index Seek / Key Lookup**
- **Index Scan** (range scan)
- **Multiple index usage** (e.g., intersection/union of rowIDs from multiple indexes)

The optimizer enumerates possible access paths for each table, then picks whichever is best. Key considerations:

- Predicate selectivity
- Sort order requirements
- Physical data layout (clustered vs. heap, presence of included columns, compression, zone maps, etc.)

### Index Include Columns

An index can store additional (included) columns in its leaf nodes. This helps queries that need those columns to avoid extra lookups (a *covering index*). For example:

```sql
CREATE INDEX idx_foo
    ON foo(a, b)
    INCLUDE (c);
```

Now, a query like:

```sql
SELECT b
  FROM foo
 WHERE a = 123
   AND c = 'WuTang';
```

might not require going back to the base table, because `c` is included in the index.

### Multiple Access Methods

Sometimes the DBMS can combine multiple indexes for a single table. Conceptually, this is like:

```
Self-Join on T
    -> Index #1 for part of the predicate
    -> Index #2 for another part
```

In PostgreSQL, this is done as a Bitmap Heap Scan or a BitmapAnd of multiple index scans.

---

## 5. Inner Joins

After deciding access paths, the next major transformation is **join ordering**. For **inner equi-joins**, standard transformations (commutativity, associativity) apply. But the optimizer must control duplication of expressions:

- Commutativity (`X ⋈ Y → Y ⋈ X`)
- Associativity (`(X ⋈ Y) ⋈ Z → X ⋈ (Y ⋈ Z)`)

In a Cascades or Volcano-like framework, the memo structure and the transformation rules ensure we do not re-visit the same sub-plan redundantly.

### Predicate Pushdown / Pullup

If a predicate references only columns from one side of a join, we can push it below the join. This can eliminate tuples earlier. However, if the predicate is expensive (e.g., `LIKE '%…'`), sometimes we might keep it above if it doesn't reduce cardinality much.

### Physical Operator Selections

Which join algorithm do we use?

- **Hash Join** if we have equi-join and enough memory for a hash table
- **Merge Join** if inputs are sorted on the join key
- **Nested-Loop Join** as fallback

---

## 6. Outer Joins

Outer joins complicate transformations because commutativity/associativity might lose or change the null-extended tuples. For instance, a left-outer join with a certain condition might not be swappable if it breaks the semantics of which side gets nulls.

### Redundancy Rule

If we detect a *null-rejecting predicate* in the query, we can convert an outer join to an inner join. Example:

```sql
SELECT S.a
  FROM S LEFT OUTER JOIN R
    ON S.id = R.id
 WHERE S.a > 10;
```

Since `S.a > 10` rejects `S.a = NULL`, any row with no match from `R` (which would produce NULL columns from `R`) is effectively discarded. So it’s safe to rewrite as an inner join.

---

## 7. Group-By Pushdown

When we have a join plus a Group-By, it may be cheaper to perform partial or complete aggregation earlier:

1. **Complete Pushdown**: If all aggregates reference columns in one table, and the group-by keys align with the primary/foreign key usage, we can push the entire Group-By below the join.
2. **Partial Pushdown**: We compute partial aggregates below the join (reducing cardinality), then finish the final aggregate above the join.

---

## 8. Special Cases: Star / Snowflake Schemas

If the optimizer detects a **star schema** or **snowflake** design (fact table + dimension tables via foreign keys), it can immediately form a left/right-deep join tree. The dimension tables are joined in an order based on selectivity, then the fact table is scanned. This avoids exploring bushy or arbitrary join orders for dimension tables.

```sql
SELECT *
  FROM fact AS F
       JOIN dim1 ON F.d1 = dim1.id
       JOIN dim2 ON F.d2 = dim2.id
       JOIN dim3 ON F.d3 = dim3.id;
```

A typical transformation is to do a chain of hash joins in a right-deep tree, scanning the fact table last.

---

## 9. Conclusion & Next Steps

It takes years to build a full set of transformations. Even top commercial systems miss transformations leading to suboptimal plans.