# PostgreSQL Planner & Parallel Execution Notes

## Planner Decision Flow

  Rows Returned   Typical Plan
  --------------- ------------------
  Very few        Index Scan
  Medium          Bitmap Heap Scan
  Large           Sequential Scan

Using `big_table1`:

``` sql
SELECT * FROM big_table1 WHERE val BETWEEN 1 AND 15;
```

Sample plan:

``` text
Index Scan using idx_big_table1_val
Index Cond: ((val >= 1) AND (val <= 15))
Index Searches: 1
Execution Time: ~15 ms
```

------------------------------------------------------------------------

``` sql
SELECT * FROM big_table1
WHERE val BETWEEN 100000 AND 150000;
```

Sample plan:

``` text
Bitmap Heap Scan
  Recheck Cond: ((val >= 100000) AND (val <= 150000))
  Rows Removed by Index Recheck: 1286650
  Heap Blocks: exact=62083 lossy=133371
    -> Bitmap Index Scan
       Index Searches: 1
Execution Time: ~4.9 s
```

------------------------------------------------------------------------

``` sql
SELECT * FROM big_table1
WHERE val BETWEEN 100000 AND 350000;
```

Sample plan:

``` text
Seq Scan
Rows Returned: ~1.25 million
Execution Time: ~7.0 s
```

## random_page_cost

Setting:

``` sql
SET random_page_cost = 1.0;
```

changed the planner from Bitmap Heap Scan to Index Scan for the
medium-selectivity query.

Observed:

``` text
Bitmap Heap Scan  -> ~4.9 s
Index Scan        -> ~19.8 s
```

Lesson: lowering `random_page_cost` too aggressively can produce a
slower plan if it no longer reflects storage characteristics.

------------------------------------------------------------------------

## effective_io_concurrency

Primary beneficiary:

-   Bitmap Heap Scan ✅
-   Regular Index Scan ❌ (generally negligible direct benefit)
-   Sequential Scan ❌

Recommended demonstration:

``` sql
SET random_page_cost = 4;
SET effective_io_concurrency = 1;
```

Run Bitmap Heap Scan, then repeat with:

``` sql
SET effective_io_concurrency = 200;
```

Compare execution times while the plan remains identical.

------------------------------------------------------------------------

# Index Searches

Range query:

``` sql
SELECT *
FROM big_table1
WHERE val BETWEEN 100000 AND 150000;
```

Plan:

``` text
Index Searches: 1
```

Reason:

One B-tree traversal, then PostgreSQL walks the linked leaf pages.

------------------------------------------------------------------------

IN predicate:

``` sql
SELECT *
FROM big_table1
WHERE val IN (10,1000,500000,900000);
```

Plan:

``` text
Index Scan
Index Searches: 4
Rows: 20
```

Each constant performs an independent B-tree search.

------------------------------------------------------------------------

# Parallel Aggregate

Example:

``` sql
EXPLAIN (ANALYZE,BUFFERS)
SELECT SUM(val)
FROM big_table1;
```

Sample plan:

``` text
Finalize Aggregate
  -> Gather
       Workers Planned: 2
       Workers Launched: 2
       -> Partial Aggregate
            -> Parallel Index Only Scan
```

Observed snippets:

``` text
Partial Aggregate
rows=1
loops=3
```

Each of the three participating processes (leader + 2 workers) produces
one partial result.

Gather:

``` text
Gather
rows=3
```

Finalize Aggregate combines the three partial sums into one final
result.

Conceptually:

``` text
Worker1 SUM
      \
Worker2 SUM ---> Finalize Aggregate ---> Final SUM
      /
Leader  SUM
```

------------------------------------------------------------------------

# Parallel AVG()

Example:

``` sql
EXPLAIN (ANALYZE,BUFFERS)
SELECT AVG(val)
FROM big_table1;
```

Plan:

``` text
Finalize Aggregate
  -> Gather
       -> Partial Aggregate
            -> Parallel Index Only Scan
```

Important concept:

Workers DO NOT send averages.

Instead each worker maintains a transition state:

``` text
Worker1
SUM = ...
COUNT = ...

Worker2
SUM = ...
COUNT = ...

Leader
SUM = ...
COUNT = ...
```

Finalize Aggregate computes:

``` text
Final SUM   = SUM1 + SUM2 + SUM3
Final COUNT = COUNT1 + COUNT2 + COUNT3

AVG = Final SUM / Final COUNT
```

This avoids incorrect results that would occur by averaging worker
averages.

Interesting observation from your plans:

``` text
SUM():
Finalize Aggregate width=8

AVG():
Finalize Aggregate width=32
```

`AVG()` requires a richer transition state (sum + count and related
metadata), so the planner reports a larger width.

------------------------------------------------------------------------

# Parallel Index Only Scan

Your aggregate queries used:

``` text
Parallel Index Only Scan
Heap Fetches: 264
Index Searches: 1
rows=1666666.67
loops=3
```

Meaning:

-   Leader + 2 workers participated.
-   Each processed approximately 1.67 million index entries.
-   Very few heap fetches were required because the visibility map
    allowed an Index Only Scan.
