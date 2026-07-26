# PostgreSQL Performance Tuning Lab Workbook

## Single Table Used Throughout

``` sql
DROP TABLE IF EXISTS big_table1;

CREATE TABLE big_table1 AS
SELECT
    i,
    (random()*1000000)::int AS val,
    repeat(md5(i::text),20) AS payload
FROM generate_series(1,5000000) i;

CREATE INDEX idx_big_table1_val ON big_table1(val);
ANALYZE big_table1;
\timing on
```

## Parameters

``` conf
random_page_cost = 1.1
effective_io_concurrency = 200

huge_pages = try
io_method = io_uring

max_worker_processes = 8
max_parallel_workers = 8
max_parallel_workers_per_gather = 4
max_parallel_maintenance_workers = 4
```

# random_page_cost

Purpose: Planner cost model.

    Default: random_page_cost=4
    SSD:     random_page_cost=1.1

``` sql
SET random_page_cost=4;
EXPLAIN (ANALYZE,BUFFERS)
SELECT * FROM big_table1 WHERE val BETWEEN 1000 AND 300000;

SET random_page_cost=1.1;
EXPLAIN (ANALYZE,BUFFERS)
SELECT * FROM big_table1 WHERE val BETWEEN 1000 AND 300000;
```

Observe planner moving between Index Scan, Bitmap Heap Scan and Parallel
Seq Scan depending on selectivity.

# effective_io_concurrency

Purpose: Improves Bitmap Heap Scan by allowing asynchronous reads.

``` sql
SET enable_seqscan=off;
SET enable_indexscan=off;

SET effective_io_concurrency=1;
EXPLAIN (ANALYZE,BUFFERS)
SELECT * FROM big_table1 WHERE val BETWEEN 100000 AND 400000;

SET effective_io_concurrency=200;
EXPLAIN (ANALYZE,BUFFERS)
SELECT * FROM big_table1 WHERE val BETWEEN 100000 AND 400000;
```

Check execution time and Heap Blocks.

# Parallel Query

``` sql
EXPLAIN (ANALYZE,BUFFERS)
SELECT count(*) FROM big_table1;
```

Expected operators:

-   Gather
-   Partial Aggregate
-   Parallel Seq Scan

Disable:

``` sql
SET max_parallel_workers_per_gather=0;
```

Run EXPLAIN again and compare.

# Huge Pages

``` conf
huge_pages=try
```

Linux:

``` bash
echo 3 > /proc/sys/vm/drop_caches
sysctl -w vm.nr_hugepages=100
grep Huge /proc/meminfo
```

Restart PostgreSQL.

``` sql
SHOW huge_pages;
SHOW huge_pages_status;
```

Disable:

``` bash
sysctl -w vm.nr_hugepages=0
```

Restart PostgreSQL and verify status becomes OFF.

Important concepts:

-   PostgreSQL page = 8 KB
-   Huge Page = 2 MB
-   1 Huge Page stores 256 PostgreSQL pages.
-   SELECT \* still reads 8 KB pages.
-   Huge Pages reduce TLB misses and page-table overhead only.

# io_method

``` conf
io_method=io_uring
```

Restart PostgreSQL.

Run:

``` sql
SELECT count(*) FROM big_table1;
```

Discuss asynchronous Linux I/O.

# Summary

  Parameter                  Planner   Executor   Restart
  -------------------------- --------- ---------- ---------
  random_page_cost           Yes       No         No
  effective_io_concurrency   No        Yes        No
  huge_pages                 No        Memory     Yes
  io_method                  No        I/O        Yes
  Parallel Workers           Yes       Yes        Yes

# Demo Flow

1.  Create table
2.  ANALYZE
3.  Baseline EXPLAIN
4.  random_page_cost
5.  effective_io_concurrency
6.  Parallel query
7.  Huge Pages enable/disable
8.  io_uring
9.  Compare optimized plans
