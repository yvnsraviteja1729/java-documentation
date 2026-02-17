<p><a target="_blank" href="https://app.eraser.io/workspace/wEiBIdcFI7KICL8TolkY" id="edit-in-eraser-github-link"><img alt="Edit in Eraser" src="https://firebasestorage.googleapis.com/v0/b/second-petal-295822.appspot.com/o/images%2Fgithub%2FOpen%20in%20Eraser.svg?alt=media&amp;token=968381c8-a7e7-472a-8ed6-4a6626da5501"></a></p>

# SQL Server Query Performance & Execution
## Overview
Query performance in SQL Server is governed by the **Query Optimizer** — a cost-based engine that evaluates hundreds of possible execution strategies and picks the one it estimates will be cheapest. Understanding how it works, what it relies on, and how to diagnose its decisions is the foundation of SQL Server performance tuning.

```
Your SQL Query
      │
      ▼
┌─────────────────────┐
│   Parser            │  Checks syntax
└────────┬────────────┘
         ▼
┌─────────────────────┐
│   Algebrizer        │  Resolves object names, data types
└────────┬────────────┘
         ▼
┌─────────────────────┐
│   Query Optimizer   │  Generates & evaluates candidate plans
│   (uses Statistics) │  Picks lowest estimated cost plan
└────────┬────────────┘
         ▼
┌─────────────────────┐
│   Plan Cache        │  Stores compiled plan for reuse
└────────┬────────────┘
         ▼
┌─────────────────────┐
│   Execution Engine  │  Runs the plan, returns results
└─────────────────────┘
```
---

## 1. Execution Plans (Estimated vs Actual)
An execution plan is a **visual roadmap** of how SQL Server executes a query — which indexes it uses, how it joins tables, the order of operations, and the estimated cost of each step.

### Estimated Execution Plan
Generated **without running the query**. SQL Server uses statistics to estimate row counts and costs. Useful when you can't afford to run an expensive query but want to check its plan.

```sql
-- Show estimated plan (Ctrl+L in SSMS, or:)
SET SHOWPLAN_XML ON;
GO
SELECT o.OrderID, c.CustomerName
FROM Orders o
JOIN Customers c ON o.CustomerID = c.CustomerID
WHERE o.OrderDate > '2024-01-01';
GO
SET SHOWPLAN_XML OFF;
```
### Actual Execution Plan
Generated **after running the query**. Contains real runtime statistics — actual row counts, actual executions, actual memory usage. This is what you use for real tuning.

```sql
-- Show actual plan (Ctrl+M in SSMS to toggle, then run query)
SET STATISTICS PROFILE ON;
GO
SELECT o.OrderID, c.CustomerName
FROM Orders o
JOIN Customers c ON o.CustomerID = c.CustomerID
WHERE o.OrderDate > '2024-01-01';
GO
SET STATISTICS PROFILE OFF;
```
### Estimated vs Actual — Key Differences
| Feature | Estimated | Actual |
| ----- | ----- | ----- |
| Query runs | ❌ No | ✅ Yes |
| Row counts | Estimated (from stats) | Real runtime counts |
| Execution count per operator | Not shown | Shown (loops!) |
| Memory grants | Estimated | Actual used vs granted |
| Warnings | Partial | Full (spills, implicit conversions) |
| Best for | Quick plan check | Real performance diagnosis |
### Reading the Execution Plan
Plans are read **right to left, top to bottom**. The rightmost operators fetch data; results flow left toward the final output.

```
Final Result ◄── Hash Match ◄── Index Seek (Orders)
▲
└── Index Seek (Customers)
```
**Key things to look for:**

- **Thick arrows** between operators = large row estimates (potential problem)
- **Warning triangles** = implicit conversions, missing indexes, spills to disk
- **Estimated vs Actual rows mismatch** = stale or missing statistics
- **Key Lookup** = missing INCLUDE columns on an index
- **Hash Match / Sort** with high cost = memory pressure or missing index
- **Parallelism (DOP)** = query went parallel, check if appropriate
```sql
-- Find expensive queries in plan cache
SELECT TOP 10
    qs.total_elapsed_time / qs.execution_count   AS avg_elapsed_time,
    qs.total_logical_reads / qs.execution_count  AS avg_logical_reads,
    qs.execution_count,
    SUBSTRING(qt.text, (qs.statement_start_offset/2)+1,
        ((CASE qs.statement_end_offset
          WHEN -1 THEN DATALENGTH(qt.text)
          ELSE qs.statement_end_offset END
          - qs.statement_start_offset)/2)+1)      AS query_text
FROM sys.dm_exec_query_stats qs
CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) qt
ORDER BY avg_elapsed_time DESC;
```
---

## 2. Statistics and How They Affect Query Plans
### What Are Statistics?
Statistics are **metadata objects** that tell the optimizer how data is distributed in a column or index. They contain a **histogram** showing the frequency of values, which the optimizer uses to estimate how many rows a filter will return (cardinality estimation).

```sql
-- View statistics for a table
SELECT name, auto_created, user_created, last_updated
FROM sys.stats
WHERE object_id = OBJECT_ID('Orders');

-- View histogram for specific statistics
DBCC SHOW_STATISTICS('Orders', 'IX_Orders_CustomerID');
```
### The Histogram
The histogram divides column values into up to **200 steps**. Each step records:

- `RANGE_HI_KEY`  — the upper boundary value of this step
- `EQ_ROWS`  — rows equal to the boundary value
- `RANGE_ROWS`  — rows within the range (between this and previous step)
- `DISTINCT_RANGE_ROWS`  — distinct values in the range
```
RANGE_HI_KEY  EQ_ROWS  RANGE_ROWS  DISTINCT_RANGE_ROWS
100           500      0           0
200           300      1200        45
500           800      2500        100
```
### How Statistics Affect Plans
The optimizer uses statistics to choose between:

- **Index Seek vs Scan** — estimated row count determines which is cheaper
- **Join algorithms** — Nested Loops (small), Hash Match (large), Merge Join (sorted)
- **Parallelism** — large estimated row counts may trigger parallel plans
- **Memory grants** — estimated size determines how much memory is pre-allocated
```sql
-- If statistics say 10 rows → Nested Loops chosen
-- If statistics say 1,000,000 rows → Hash Match chosen
-- Wrong statistics → wrong join → poor performance
```
### Stale Statistics — The Silent Killer
Statistics are automatically updated when **~20% of rows change** (auto update threshold). For large tables, this threshold may never be crossed even after millions of changes, leaving the optimizer working with outdated histograms.

```sql
-- Manually update statistics
UPDATE STATISTICS Orders;                          -- all stats on table
UPDATE STATISTICS Orders IX_Orders_CustomerID;     -- specific index stats
UPDATE STATISTICS Orders WITH FULLSCAN;            -- full scan, most accurate
UPDATE STATISTICS Orders WITH SAMPLE 30 PERCENT;  -- sampled, faster

-- Update all stats in database
EXEC sp_updatestats;
```
```sql
-- Find stale statistics
SELECT
    OBJECT_NAME(s.object_id)    AS TableName,
    s.name                      AS StatName,
    sp.last_updated,
    sp.rows,
    sp.rows_sampled,
    sp.modification_counter
FROM sys.stats s
CROSS APPLY sys.dm_db_stats_properties(s.object_id, s.stats_id) sp
WHERE sp.modification_counter > 1000
ORDER BY sp.modification_counter DESC;
```
### Cardinality Estimator Versions
SQL Server has two cardinality estimator (CE) versions:

- **Legacy CE** (SQL Server 2012 and earlier, CE version 70)
- **New CE** (SQL Server 2014+, CE version 120/130/150)
```sql
-- Force legacy CE for a specific query (troubleshooting)
SELECT * FROM Orders
OPTION (USE HINT('FORCE_LEGACY_CARDINALITY_ESTIMATION'));
```
---

## 3. Parameter Sniffing
### What Is Parameter Sniffing?
When SQL Server first compiles a stored procedure or parameterized query, it **sniffs** (inspects) the parameter values at that moment and builds a plan optimized for those specific values. This plan is then **cached and reused** for all future executions — even if future executions use very different parameter values.

```sql
CREATE PROCEDURE GetOrders @CustomerID INT
AS
    SELECT * FROM Orders WHERE CustomerID = @CustomerID;
GO
```
```
First call: EXEC GetOrders @CustomerID = 101
  → Customer 101 has 5 orders
  → Optimizer chooses: Index Seek (perfect for 5 rows)
  → Plan cached ✅
Second call: EXEC GetOrders @CustomerID = 999
  → Customer 999 has 2,000,000 orders
  → Reuses cached plan: Index Seek + 2M Key Lookups 💀
  → Should have used Table Scan / Hash Join instead
```
### Detecting Parameter Sniffing
```sql
-- Compare estimated vs actual rows in execution plan
-- Large discrepancy = parameter sniffing or stale stats

-- Check cached plan for a stored procedure
SELECT
    qs.execution_count,
    qs.total_logical_reads,
    qs.total_elapsed_time,
    qp.query_plan,
    qt.text
FROM sys.dm_exec_procedure_stats ps
CROSS APPLY sys.dm_exec_sql_text(ps.sql_handle) qt
CROSS APPLY sys.dm_exec_query_plan(ps.plan_handle) qp
JOIN sys.dm_exec_query_stats qs ON ps.plan_handle = qs.plan_handle
WHERE OBJECT_NAME(ps.object_id) = 'GetOrders';
```
### Solutions for Parameter Sniffing
**Option 1: OPTIMIZE FOR** — Compile the plan for a specific "typical" value

```sql
CREATE PROCEDURE GetOrders @CustomerID INT
AS
    SELECT * FROM Orders WHERE CustomerID = @CustomerID
    OPTION (OPTIMIZE FOR (@CustomerID = 101)); -- compile for this value

-- Or optimize for unknown (average statistics)
    OPTION (OPTIMIZE FOR (@CustomerID UNKNOWN));
```
**Option 2: WITH RECOMPILE** — Recompile plan on every execution (expensive but always optimal)

```sql
CREATE PROCEDURE GetOrders @CustomerID INT
WITH RECOMPILE   -- recompile entire procedure every time
AS
    SELECT * FROM Orders WHERE CustomerID = @CustomerID;

-- Or recompile just the statement
    SELECT * FROM Orders WHERE CustomerID = @CustomerID
    OPTION (RECOMPILE);  -- preferred — only recompiles this statement
```
**Option 3: Local Variables** — Copy parameter to local variable (optimizer can't sniff it)

```sql
CREATE PROCEDURE GetOrders @CustomerID INT
AS
    DECLARE @LocalID INT = @CustomerID;  -- optimizer uses average stats
    SELECT * FROM Orders WHERE CustomerID = @LocalID;
```
**Option 4: Multiple Plans** — Split into separate procedures for different data shapes

```sql
-- Separate procedure for high-volume customers
IF @CustomerID IN (SELECT CustomerID FROM HighVolumeCustomers)
    EXEC GetOrdersLargeCustomer @CustomerID;
ELSE
    EXEC GetOrdersSmallCustomer @CustomerID;
```
### Summary of Solutions
| Solution | When to Use | Tradeoff |
| ----- | ----- | ----- |
|  | General skewed distributions | Uses average stats — may not be optimal |
|  | Known "typical" parameter value | Hardcoded assumption |
|  | Highly variable data, infrequent calls | CPU cost per execution |
| Local variables | Simple fix for moderate skew | Loses all parameter info |
| Query Store | SQL 2016+, managed fix | Best long-term solution |
---

## 4. Query Hints
Query hints **override the optimizer's decisions**. Use them sparingly — they fix one scenario but may break others as data changes.

### Table Hints
**NOLOCK (Read Uncommitted)**

```sql
-- Read without acquiring shared locks — dirty reads possible
SELECT * FROM Orders WITH (NOLOCK);

-- Equivalent to:
SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
SELECT * FROM Orders;
```
>  ⚠️ NOLOCK can return duplicate rows, missing rows, or rows mid-update. Never use on financial or critical data. 

**ROWLOCK / PAGELOCK / TABLOCK**

```sql
-- Force row-level locks (prevent page/table lock escalation)
UPDATE Orders WITH (ROWLOCK)
SET Status = 'Processed'
WHERE OrderID = 5000;

-- Force table lock (for bulk operations)
INSERT INTO StagingTable WITH (TABLOCK)
SELECT * FROM SourceTable;
```
**FORCESEEK / FORCESCAN**

```sql
-- Force an index seek (even if optimizer prefers scan)
SELECT * FROM Orders WITH (FORCESEEK)
WHERE CustomerID = 101;

-- Force a specific index
SELECT * FROM Orders WITH (INDEX(IX_Orders_CustomerID))
WHERE CustomerID = 101;

-- Force a scan
SELECT * FROM Orders WITH (FORCESCAN);
```
### Query Hints (OPTION clause)
**RECOMPILE** — Force fresh plan compilation for this execution

```sql
SELECT * FROM Orders
WHERE CustomerID = @CustomerID
OPTION (RECOMPILE);
```
**OPTIMIZE FOR** — Compile plan for a specific parameter value

```sql
SELECT * FROM Orders
WHERE CustomerID = @CustomerID
OPTION (OPTIMIZE FOR (@CustomerID = 101));

SELECT * FROM Orders
WHERE CustomerID = @CustomerID
OPTION (OPTIMIZE FOR (@CustomerID UNKNOWN));
```
**FORCE ORDER** — Force joins to execute in the order written in the query

```sql
-- Optimizer normally reorders joins freely
-- FORCE ORDER overrides this
SELECT o.OrderID, c.CustomerName, p.ProductName
FROM Orders o
JOIN Customers c ON o.CustomerID = c.CustomerID
JOIN Products p ON o.ProductID = p.ProductID
OPTION (FORCE ORDER);  -- joins happen in this exact order
```
>  ⚠️ FORCE ORDER can hurt performance if your written order is suboptimal. Use only when you're certain your join order is better than the optimizer's. 

**MAXDOP** — Override degree of parallelism for a query

```sql
-- Force serial execution (no parallelism)
SELECT * FROM LargeTable
OPTION (MAXDOP 1);

-- Limit to 4 parallel threads
SELECT * FROM LargeTable
OPTION (MAXDOP 4);
```
**USE HINT** — Modern, cleaner hint syntax (SQL Server 2016+)

```sql
SELECT * FROM Orders
OPTION (
    USE HINT('DISABLE_OPTIMIZED_NESTED_LOOP'),
    USE HINT('FORCE_LEGACY_CARDINALITY_ESTIMATION')
);
```
**HASH JOIN / MERGE JOIN / LOOP JOIN** — Force a specific join algorithm

```sql
SELECT * FROM Orders o
JOIN Customers c ON o.CustomerID = c.CustomerID
OPTION (HASH JOIN);   -- force hash match

OPTION (MERGE JOIN);  -- force merge join (both inputs must be sorted)
OPTION (LOOP JOIN);   -- force nested loops
```
### Hint Decision Guide
| Problem | Hint to Consider |
| ----- | ----- |
| Blocking on reporting queries |  |
| Bad plan due to parameter sniffing | <p></p><p></p> |
| Optimizer choosing wrong index |  |
| Wrong join algorithm chosen |  |
| Parallel plan causing contention |  |
| Wrong join order |  |
---

## 5. SET STATISTICS IO and SET STATISTICS TIME
These are diagnostic commands that reveal exactly how much work SQL Server did to execute a query.

### SET STATISTICS IO
Reports **page-level I/O** for every table touched by the query.

```sql
SET STATISTICS IO ON;

SELECT o.OrderID, c.CustomerName
FROM Orders o
JOIN Customers c ON o.CustomerID = c.CustomerID
WHERE o.OrderDate > '2024-01-01';

SET STATISTICS IO OFF;
```
**Output:**

```
Table 'Customers'. Scan count 1, logical reads 8,
                   physical reads 0, read-ahead reads 0,
                   lob logical reads 0
Table 'Orders'.    Scan count 1, logical reads 1854,
                   physical reads 12, read-ahead reads 1830,
                   lob logical reads 0
```
**What each field means:**

| Field | Meaning | Tuning Signal |
| ----- | ----- | ----- |
|  | Number of index/table scans | > 1 in a loop = nested loop concern |
|  | Pages read from buffer pool | <p>**Primary tuning metric**</p><p> — reduce this</p> |
|  | Pages read from disk (cache miss) | High = buffer pool too small or cold |
|  | Pages prefetched by read-ahead | High on scans = normal and good |
|  | Pages for LOB data (text, image, XML) | High = avoid selecting large LOB columns |
### SET STATISTICS TIME
Reports **CPU and elapsed time** broken into parse, compile, and execution phases.

```sql
SET STATISTICS TIME ON;
SELECT * FROM Orders WHERE CustomerID = 101;
SET STATISTICS TIME OFF;
```
**Output:**

```
SQL Server parse and compile time:
   CPU time = 0 ms, elapsed time = 8 ms.
SQL Server Execution Times:
   CPU time = 16 ms,  elapsed time = 23 ms.
```
**Interpreting the output:**

- **High compile time** relative to execution = plan compilation overhead (consider caching)
- **CPU time ≈ elapsed time** = query is CPU-bound (single threaded or efficient parallel)
- **CPU time << elapsed time** = query is waiting (I/O-bound, blocking, network)
- **CPU time >> elapsed time** = parallel execution (multiple threads, sum of all thread CPU)
### Using Both Together
```sql
SET STATISTICS IO ON;
SET STATISTICS TIME ON;

-- Your query here

SET STATISTICS IO OFF;
SET STATISTICS TIME OFF;
```
>  **Tip:** Always run `SET STATISTICS IO ON` before tuning a query. If you reduce logical reads, you've genuinely improved the query. Elapsed time alone can be misleading due to caching effects. 

---

## 6. SARGability
### What is SARGability?
SARG stands for **Search ARGument**. A SARGable predicate is one that allows SQL Server to use an **index seek** — it can navigate directly to matching rows in the B-Tree. A non-SARGable predicate forces a full scan.

The rule: **the indexed column must appear alone on one side of the comparison, without being wrapped in a function or expression.**

### Non-SARGable vs SARGable Patterns
**Functions on columns — most common mistake**

```sql
-- ❌ Non-SARGable — function applied to column, can't seek
SELECT * FROM Orders WHERE YEAR(OrderDate) = 2024;
SELECT * FROM Orders WHERE MONTH(OrderDate) = 6;
SELECT * FROM Employees WHERE UPPER(LastName) = 'SMITH';
SELECT * FROM Orders WHERE DATEPART(dw, OrderDate) = 2;

-- ✅ SARGable rewrites
SELECT * FROM Orders
WHERE OrderDate >= '2024-01-01' AND OrderDate < '2025-01-01';

SELECT * FROM Orders
WHERE OrderDate >= '2024-06-01' AND OrderDate < '2024-07-01';

SELECT * FROM Employees
WHERE LastName = 'Smith';  -- use a case-insensitive collation instead

SELECT * FROM Orders
WHERE OrderDate >= '2024-01-08' AND OrderDate < '2024-01-09'; -- Monday
```
**Arithmetic on columns**

```sql
-- ❌ Non-SARGable — math applied to column
SELECT * FROM Products WHERE Price * 1.1 > 100;
SELECT * FROM Accounts WHERE Balance + Interest > 5000;

-- ✅ SARGable — move math to the value side
SELECT * FROM Products WHERE Price > 100 / 1.1;
SELECT * FROM Accounts WHERE Balance > 5000 - Interest;
```
**Leading wildcard in LIKE**

```sql
-- ❌ Non-SARGable — leading wildcard forces full scan
SELECT * FROM Customers WHERE LastName LIKE '%Smith';
SELECT * FROM Products WHERE Description LIKE '%wheel%';

-- ✅ SARGable — trailing wildcard only
SELECT * FROM Customers WHERE LastName LIKE 'Smith%';

-- For contains search, consider Full-Text Search instead
SELECT * FROM Products WHERE CONTAINS(Description, 'wheel');
```
**ISNULL / COALESCE on columns**

```sql
-- ❌ Non-SARGable
SELECT * FROM Orders WHERE ISNULL(ShipDate, '1900-01-01') = '1900-01-01';

-- ✅ SARGable
SELECT * FROM Orders WHERE ShipDate IS NULL;
```
**Type mismatch (see Implicit Conversions section)**

```sql
-- ❌ Non-SARGable — implicit conversion on column
SELECT * FROM Orders WHERE CustomerID = '101';  -- CustomerID is INT

-- ✅ SARGable — matching types
SELECT * FROM Orders WHERE CustomerID = 101;
```
### The SARGability Test
Ask yourself: _"Can SQL Server look this value up in the B-Tree without reading every row?"_

If the answer involves evaluating a function on every row to check the condition — it's not SARGable.

---

## 7. Implicit Conversions and Their Performance Impact
### What Is an Implicit Conversion?
When you compare two values of different data types, SQL Server must convert one to match the other. If it converts the **column** (not the literal), it must evaluate the function on every row — making the predicate non-SARGable and preventing index seeks.

```sql
-- Column is INT, but value is VARCHAR
SELECT * FROM Orders WHERE CustomerID = '101';

-- SQL Server internally rewrites this as:
SELECT * FROM Orders WHERE CONVERT(VARCHAR, CustomerID) = '101';
-- Now it can't seek — must scan every row
```
### Data Type Precedence
SQL Server converts the **lower-precedence type** to the higher one. The full precedence order (high to low) includes:

```
datetime > float > decimal > int > varchar > nvarchar ...
```
This means if you compare `INT` column with `VARCHAR` literal, the VARCHAR is converted — **good**, seek still works. But if you compare `VARCHAR` column with `NVARCHAR` literal, the `VARCHAR` column gets converted — **bad**, seek breaks.

```sql
-- VARCHAR column vs NVARCHAR literal (N prefix)
-- BAD: column gets converted, seek is lost
SELECT * FROM Customers WHERE LastName = N'Smith';  -- LastName is VARCHAR

-- GOOD: matching types
SELECT * FROM Customers WHERE LastName = 'Smith';   -- no N prefix
```
### Finding Implicit Conversions
**In execution plans:** Look for yellow warning triangles with "Type conversion in expression may affect CardinalityEstimate" or "CONVERT_IMPLICIT" in the plan XML.

```sql
-- Find queries with implicit conversion warnings in plan cache
SELECT
    qp.query_plan,
    qt.text AS query_text
FROM sys.dm_exec_cached_plans cp
CROSS APPLY sys.dm_exec_query_plan(cp.plan_handle) qp
CROSS APPLY sys.dm_exec_sql_text(cp.sql_handle) qt
WHERE CAST(qp.query_plan AS NVARCHAR(MAX))
      LIKE '%CONVERT_IMPLICIT%';
```
### Common Implicit Conversion Scenarios
| Scenario | Column Type | Value/Param Type | Impact |
| ----- | ----- | ----- | ----- |
| ORM passes string for int FK |  |  | 🟡 Column not converted, seek OK but bad habit |
| VARCHAR column vs NVARCHAR param |  |  | 🔴 Column converted, seek lost |
| DATETIME vs DATE comparison |  |  | 🟡 Usually safe |
| INT vs BIGINT |  |  | 🟡 Usually safe |
| ORM sends NVARCHAR for VARCHAR PK |  |  | 🔴 Column converted, seek lost |
| Numeric string vs INT |  |  | 🟡 Literal converted, seek OK |
### Prevention
```sql
-- Always match parameter types to column types
-- In stored procedures:
CREATE PROCEDURE GetCustomer @CustomerID INT  -- match column type exactly
AS
    SELECT * FROM Customers WHERE CustomerID = @CustomerID;

-- In application code — use strongly typed parameters:
-- cmd.Parameters.Add("@CustomerID", SqlDbType.Int).Value = customerId;
-- NOT:
-- cmd.Parameters.AddWithValue("@CustomerID", "101");  ← bad
```
---

## 8. Optimizer and Plan Cache
### How the Query Optimizer Works
The Query Optimizer is a **cost-based optimizer (CBO)**. It doesn't find the perfect plan — it finds a "good enough" plan within a time budget by evaluating a subset of all possible plans.

```
Input Query
    │
    ▼
Trivial Plan Check ──── Simple query? → Use trivial plan (fast path)
    │ No
    ▼
Exploration Phase
  - Generate candidate plans (join orders, index choices, etc.)
  - Estimate cost of each using statistics
  - Apply transformation rules
    │
    ▼
Cost-Based Selection
  - Compare estimated costs
  - Select lowest-cost plan
    │
    ▼
Compiled Plan → Stored in Plan Cache
```
### The Plan Cache
The plan cache stores **compiled execution plans** in memory so they can be reused without recompilation. Plan reuse saves significant CPU time — plan compilation can be expensive.

```sql
-- View what's in the plan cache
SELECT
    usecounts,
    size_in_bytes,
    cacheobjtype,
    objtype,
    text AS query_text
FROM sys.dm_exec_cached_plans cp
CROSS APPLY sys.dm_exec_sql_text(cp.plan_handle)
WHERE objtype IN ('Proc', 'Prepared', 'Adhoc')
ORDER BY usecounts DESC;

-- Check plan cache memory usage
SELECT
    SUM(size_in_bytes) / 1024 / 1024 AS cache_size_mb,
    COUNT(*)                          AS cached_plans,
    objtype
FROM sys.dm_exec_cached_plans
GROUP BY objtype
ORDER BY cache_size_mb DESC;
```
### Plan Cache Invalidation
A cached plan is discarded (invalidated) when:

- **Statistics are updated** — may trigger recompile
- **Schema changes** — table/index altered or dropped
- **SET options change** — different ANSI settings produce different plans
- **Explicit cache clear** — DBCC FREEPROCCACHE
- **Memory pressure** — SQL Server evicts least-used plans
- **WITH RECOMPILE** is used
```sql
-- Clear entire plan cache (⚠️ never run on production without care)
DBCC FREEPROCCACHE;

-- Clear a single plan from cache
DBCC FREEPROCCACHE (plan_handle);

-- Clear cache for a specific database
DBCC FLUSHPROCINDB (database_id);
```
### Ad Hoc vs Parameterized Plans
**Ad hoc queries** (literal values in query text) each get their own plan — even if they're logically identical:

```sql
-- Each generates a separate cached plan — cache bloat!
SELECT * FROM Orders WHERE CustomerID = 101;
SELECT * FROM Orders WHERE CustomerID = 102;
SELECT * FROM Orders WHERE CustomerID = 103;
```
**Parameterized queries** share a single cached plan:

```sql
-- One cached plan, reused for all CustomerID values
SELECT * FROM Orders WHERE CustomerID = @CustomerID;
```
**Optimize for Ad Hoc Workloads** — stores only a stub on first execution, full plan on second. Reduces cache bloat from single-use queries.

```sql
-- Enable at server level (recommended for most servers)
EXEC sp_configure 'optimize for ad hoc workloads', 1;
RECONFIGURE;
```
**Forced Parameterization** — SQL Server automatically parameterizes all queries in a database:

```sql
ALTER DATABASE MyDB SET PARAMETERIZATION FORCED;
```
### Query Store (SQL Server 2016+)
Query Store is the modern replacement for manual plan cache analysis. It **persists** query plans and runtime statistics across restarts, letting you track plan changes over time and force specific plans.

```sql
-- Enable Query Store
ALTER DATABASE MyDB SET QUERY_STORE = ON;

-- Find top 10 queries by total CPU
SELECT TOP 10
    qt.query_sql_text,
    qrs.avg_cpu_time,
    qrs.avg_logical_io_reads,
    qrs.count_executions,
    qp.query_plan
FROM sys.query_store_query q
JOIN sys.query_store_query_text qt ON q.query_text_id = qt.query_text_id
JOIN sys.query_store_plan qp       ON q.query_id = qp.query_id
JOIN sys.query_store_runtime_stats qrs ON qp.plan_id = qrs.plan_id
ORDER BY qrs.avg_cpu_time DESC;

-- Force a specific plan (fix for parameter sniffing regression)
EXEC sys.sp_query_store_force_plan
    @query_id = 42,
    @plan_id = 7;
```
### Optimizer Limitations to Know
**Optimizer timeout** — For very complex queries, the optimizer stops exploring after a time budget and uses the best plan found so far. You may see "Good Enough Plan Found" in the plan.

**Row goal optimization** — For TOP, EXISTS, FAST N hints, the optimizer optimizes for finding the first N rows quickly rather than the total cost of all rows.

```sql
-- Optimizer targets getting first 10 rows fast, not full scan
SELECT TOP 10 * FROM Orders ORDER BY OrderDate DESC;
```
**Cardinality estimation errors** — When statistics are stale or the query has complex predicates (especially with OR, multiple joins, or correlated columns), row estimates can be wildly off — leading to poor plan choices.

---

## Putting It All Together — Tuning Workflow
```
Query is slow
      │
      ▼
1. SET STATISTICS IO ON → check logical reads
      │
      ▼
2. View Actual Execution Plan
      │
      ├── Missing Index warning?        → Create the suggested index
      ├── Key Lookup?                   → Add INCLUDE columns
      ├── Estimated ≠ Actual rows?      → Update statistics
      ├── Implicit conversion warning?  → Fix parameter/column type mismatch
      ├── Table Scan on large table?    → Check SARGability of WHERE clause
      ├── Parameter sniffing suspected? → OPTION (RECOMPILE) or OPTIMIZE FOR
      └── Wrong join algorithm?         → Consider join hints (last resort)
      │
      ▼
3. Verify improvement with STATISTICS IO
   (logical reads should decrease)
      │
      ▼
4. Monitor with Query Store over time
```
---

## Quick Reference
| Tool / Feature | Purpose |
| ----- | ----- |
| Estimated Execution Plan | Check plan without running query |
| Actual Execution Plan | Diagnose real performance, see actual vs estimated rows |
|  | Measure logical/physical reads per table |
|  | Measure CPU and elapsed time |
| Statistics | Drive optimizer cardinality estimates |
|  | Refresh stale histograms |
| Parameter Sniffing | Cached plan optimized for wrong parameter values |
|  | Force fresh plan each execution |
|  | Compile plan for specific/average value |
| SARGability | Predicate must allow index seek (no functions on columns) |
| Implicit Conversions | Type mismatch causes column conversion, kills seeks |
| Plan Cache | Stores compiled plans for reuse |
| Query Store | Persists plan history, enables plan forcing |
|  | Clear plan cache (use with care) |
|  | Read without locks — accepts dirty reads |
|  | Control parallelism per query |




<!--- Eraser file: https://app.eraser.io/workspace/wEiBIdcFI7KICL8TolkY --->