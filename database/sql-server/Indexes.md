<p><a target="_blank" href="https://app.eraser.io/workspace/C05GfTfsKeIoc8NBflEl" id="edit-in-eraser-github-link"><img alt="Edit in Eraser" src="https://firebasestorage.googleapis.com/v0/b/second-petal-295822.appspot.com/o/images%2Fgithub%2FOpen%20in%20Eraser.svg?alt=media&amp;token=968381c8-a7e7-472a-8ed6-4a6626da5501"></a></p>

# SQL Server Indexes
## Overview
An index in SQL Server is a **data structure that speeds up data retrieval** on a table at the cost of additional storage and write overhead. Think of it like a book's index — instead of scanning every page, you jump directly to what you need.

Without indexes, SQL Server must scan every row in a table (**Table Scan**). With the right indexes, it jumps directly to the relevant data (**Index Seek**).

```
Without Index:          With Index:
┌──────────┐           ┌─────────────┐     ┌──────────┐
│ Row 1    │ ← scan    │ Index Entry │────►│ Row 847  │
│ Row 2    │ ← scan    │ (sorted)    │     └──────────┘
│ ...      │ ← scan    └─────────────┘
│ Row 847  │ ← found!
│ ...      │ ← still scanning
└──────────┘
```
---

## 1. Clustered vs Non-Clustered Index
### Clustered Index
A clustered index **defines the physical order** of data in the table. The table's data rows are stored sorted by the clustered index key. Because data can only be physically sorted one way, **a table can have only one clustered index**.

```sql
-- Creating a clustered index
CREATE CLUSTERED INDEX IX_Orders_OrderID
ON Orders (OrderID);
```
- The leaf level of a clustered index **is the actual data** (the table itself)
- By default, a PRIMARY KEY creates a clustered index
- Data rows are stored in the B-Tree leaf pages directly
```
Clustered Index B-Tree Structure:

         [Root Page]
        /     |      \
   [500]    [1000]   [1500]       ← Intermediate Pages
   /   \    /    \    /   \
[1-499][500-999][1000-1499][1500+] ← Leaf Pages = Actual Data Rows
```
### Non-Clustered Index
A non-clustered index is a **separate structure** from the data. It contains the index key columns plus a pointer back to the actual data row (via the clustered index key or a Row ID if no clustered index exists).

```sql
-- Creating a non-clustered index
CREATE NONCLUSTERED INDEX IX_Orders_CustomerID
ON Orders (CustomerID);
```
- A table can have **up to 999 non-clustered indexes**
- The leaf level contains index keys + **row locators** (pointer to actual data)
- After finding the key in the index, SQL Server does a **Key Lookup** to fetch remaining columns from the clustered index
```
Non-Clustered Index Structure:

Non-Clustered Index          Clustered Index (Table)
┌─────────────────┐          ┌──────────────────────┐
│ CustomerID │ →  │─────────►│ OrderID (Clustered)  │
│ 101        │ →  │          │ CustomerID           │
│ 102        │ →  │          │ OrderDate            │
│ 103        │ →  │          │ Amount               │
└─────────────────┘          └──────────────────────┘
(Separate Structure)         (Physical Data)
```
### Key Differences
| Feature | Clustered | Non-Clustered |
| ----- | ----- | ----- |
| Number per table | **1 only** | Up to 999 |
| Leaf level contains | Actual data rows | Pointers to data |
| Storage | No extra storage (IS the table) | Extra storage required |
| Read speed | Faster for range scans | Requires key lookup |
| Write overhead | Higher (data reorder) | Lower |
| Default for PRIMARY KEY | Yes | No |
---

## 2. Composite Indexes and Key Column Order
A composite index (also called a compound index) is an index on **two or more columns**. The order of columns in a composite index is critically important.

```sql
-- Composite index on (LastName, FirstName, DateOfBirth)
CREATE NONCLUSTERED INDEX IX_Employees_Name
ON Employees (LastName, FirstName, DateOfBirth);
```
### The Leftmost Prefix Rule
SQL Server can use a composite index only if the query filters on the **leading (leftmost) columns** of the index.

```sql
-- ✅ Can use IX_Employees_Name (filters on LastName — leftmost)
SELECT * FROM Employees WHERE LastName = 'Smith';

-- ✅ Can use IX_Employees_Name (filters on LastName + FirstName)
SELECT * FROM Employees WHERE LastName = 'Smith' AND FirstName = 'John';

-- ✅ Can use IX_Employees_Name (all three columns)
SELECT * FROM Employees
WHERE LastName = 'Smith' AND FirstName = 'John' AND DateOfBirth = '1990-01-01';

-- ❌ Cannot use IX_Employees_Name (skips LastName — leftmost column)
SELECT * FROM Employees WHERE FirstName = 'John';

-- ❌ Cannot use IX_Employees_Name efficiently (skips LastName)
SELECT * FROM Employees WHERE DateOfBirth = '1990-01-01';
```
### Column Order Best Practices
**Equality columns first, range columns last.** If your query filters `Status = 'Active'` (equality) and `OrderDate > '2024-01-01'` (range), put Status first.

```sql
-- ✅ Good — equality first, range last
CREATE INDEX IX_Orders_StatusDate ON Orders (Status, OrderDate);

-- ❌ Less optimal — range first blocks use of second column
CREATE INDEX IX_Orders_DateStatus ON Orders (OrderDate, Status);
```
**High selectivity columns first.** Put the column that filters out the most rows first (e.g., CustomerID before Status if CustomerID is more selective).

---

## 3. Covering Indexes and INCLUDE Columns
### The Key Lookup Problem
When a non-clustered index is used but the query needs columns not in the index, SQL Server must do a **Key Lookup** — going back to the clustered index to fetch the missing columns. For large result sets, this becomes expensive.

```sql
CREATE NONCLUSTERED INDEX IX_Orders_CustomerID
ON Orders (CustomerID);

-- This causes a Key Lookup for OrderDate and Amount
-- (they're not in the index)
SELECT CustomerID, OrderDate, Amount
FROM Orders
WHERE CustomerID = 101;
```
### Covering Index
A **covering index** includes all columns needed by the query, eliminating the Key Lookup entirely.

```sql
-- Option 1: Add extra columns as key columns
CREATE NONCLUSTERED INDEX IX_Orders_CustomerID
ON Orders (CustomerID, OrderDate, Amount);

-- Option 2: Use INCLUDE (preferred)
CREATE NONCLUSTERED INDEX IX_Orders_CustomerID
ON Orders (CustomerID)
INCLUDE (OrderDate, Amount);
```
### Key Columns vs INCLUDE Columns
| Feature | Key Columns | INCLUDE Columns |
| ----- | ----- | ----- |
| Sorted in index | ✅ Yes | ❌ No |
| Used for filtering/sorting | ✅ Yes | ❌ No (only for covering) |
| Affect index size | Moderate | Can add many without overhead |
| Available in index seek | ✅ Yes | ✅ Yes (at leaf level only) |
```sql
-- Best practice: Only key the columns you filter/sort on,
-- INCLUDE the columns you SELECT
CREATE NONCLUSTERED INDEX IX_Orders_Covering
ON Orders (CustomerID, OrderDate)    -- filter and sort columns
INCLUDE (Amount, ShipAddress, Status); -- just need to read these
```
>  **Rule of thumb:** If you see **Key Lookup** operators in your execution plan, consider adding an INCLUDE to make the index covering. 

---

## 4. Filtered Indexes
A filtered index is a non-clustered index with a **WHERE clause** — it only indexes a subset of rows. This makes it smaller, faster to maintain, and more efficient for queries targeting that subset.

```sql
-- Only index active orders (not completed/cancelled)
CREATE NONCLUSTERED INDEX IX_Orders_Active
ON Orders (CustomerID, OrderDate)
WHERE Status = 'Active';

-- Only index non-null values
CREATE NONCLUSTERED INDEX IX_Employees_Manager
ON Employees (ManagerID)
WHERE ManagerID IS NOT NULL;
```
### When to Use Filtered Indexes
**Sparse columns** — A column that is NULL for most rows but queried when not NULL (e.g., `DeletedAt`, `ProcessedDate`).

**Status-based queries** — You frequently query only `Status = 'Pending'` rows in a table where 95% of rows are `Status = 'Completed'`.

**Soft deletes** — Tables with an `IsDeleted` flag where you almost always query `IsDeleted = 0`.

```sql
-- Soft delete pattern — index only non-deleted rows
CREATE NONCLUSTERED INDEX IX_Customers_Active
ON Customers (Email)
WHERE IsDeleted = 0;
```
### Benefits vs Regular Index
|  | Regular Index | Filtered Index |
| ----- | ----- | ----- |
| Index size | Full table | Subset only (much smaller) |
| Maintenance cost | Higher | Lower (fewer rows to update) |
| Statistics quality | Diluted | More accurate for that subset |
| Applicability | All queries | Only queries matching the filter |
>  **Gotcha:** The query's WHERE clause must be compatible with the filter predicate for SQL Server to use the filtered index. 

---

## 5. Index Fragmentation — Rebuild vs Reorganize
### What is Fragmentation?
When data is inserted, updated, or deleted, index pages can become fragmented in two ways:

**Internal Fragmentation (Page Density)** — Pages have lots of empty space because rows were deleted or rows are small. Storage is wasted.

**External Fragmentation (Logical Ordering)** — The logical order of pages doesn't match their physical order on disk. Sequential reads become random reads — hurting read-ahead performance.

```sql
-- Check fragmentation level
SELECT
    OBJECT_NAME(ips.object_id)      AS TableName,
    i.name                          AS IndexName,
    ips.index_type_desc,
    ips.avg_fragmentation_in_percent,
    ips.page_count
FROM sys.dm_db_index_physical_stats(
    DB_ID(), NULL, NULL, NULL, 'LIMITED') ips
JOIN sys.indexes i
    ON ips.object_id = i.object_id
    AND ips.index_id = i.index_id
WHERE ips.avg_fragmentation_in_percent > 10
ORDER BY ips.avg_fragmentation_in_percent DESC;
```
### Reorganize
`ALTER INDEX ... REORGANIZE` is an **online, incremental** operation. It defragments the leaf level of the index by physically reordering the leaf pages. It's always online (doesn't lock the table) and can be stopped and restarted.

```sql
ALTER INDEX IX_Orders_CustomerID ON Orders REORGANIZE;
```
### Rebuild
`ALTER INDEX ... REBUILD` **drops and recreates** the index entirely. It's more thorough — it fixes both internal and external fragmentation and updates statistics with a full scan. By default it's offline (locks the table), but Enterprise Edition supports online rebuilds.

```sql
-- Offline rebuild (all editions)
ALTER INDEX IX_Orders_CustomerID ON Orders REBUILD;

-- Online rebuild (Enterprise Edition only)
ALTER INDEX IX_Orders_CustomerID ON Orders
REBUILD WITH (ONLINE = ON);

-- Rebuild all indexes on a table
ALTER INDEX ALL ON Orders REBUILD;
```
### When to Use Which
| Fragmentation Level | Action | Reason |
| ----- | ----- | ----- |
| < 10% | Do nothing | Overhead not worth it |
| 10% – 30% | REORGANIZE | Light fix, online, low impact |
| > 30% | REBUILD | Full reconstruction needed |
| Any level, tiny table (< 1000 pages) | Do nothing | Fragmentation impact is negligible |
```sql
-- Automated maintenance script
IF @fragmentation BETWEEN 10 AND 30
    ALTER INDEX @IndexName ON @TableName REORGANIZE;
ELSE IF @fragmentation > 30
    ALTER INDEX @IndexName ON @TableName REBUILD;
```
>  **Note:** REBUILD resets and updates statistics automatically. REORGANIZE does NOT update statistics — run `UPDATE STATISTICS` separately afterward. 

---

## 6. Missing Index DMVs
SQL Server's query optimizer tracks **potential indexes** that would have improved query performance. These are exposed through Dynamic Management Views (DMVs).

```sql
-- Find missing indexes with impact score
SELECT TOP 20
    ROUND(migs.avg_total_user_cost *
          migs.avg_user_impact *
          (migs.user_seeks + migs.user_scans), 0) AS [Total Impact Score],
    migs.avg_user_impact                           AS [Avg % Improvement],
    migs.user_seeks                                AS [Seeks],
    migs.user_scans                                AS [Scans],
    mid.statement                                  AS [Table],
    mid.equality_columns,
    mid.inequality_columns,
    mid.included_columns,
    -- Suggested CREATE INDEX statement
    'CREATE NONCLUSTERED INDEX IX_Missing_' +
    REPLACE(REPLACE(mid.statement, '[', ''), ']', '') + '_' +
    CAST(mig.index_group_handle AS VARCHAR) +
    ' ON ' + mid.statement +
    ' (' + ISNULL(mid.equality_columns, '') +
    CASE WHEN mid.inequality_columns IS NOT NULL
         THEN (CASE WHEN mid.equality_columns IS NOT NULL
               THEN ', ' ELSE '' END) + mid.inequality_columns
         ELSE '' END + ')' +
    ISNULL(' INCLUDE (' + mid.included_columns + ')', '') AS [Suggested Index]
FROM sys.dm_db_missing_index_groups mig
JOIN sys.dm_db_missing_index_group_stats migs
    ON mig.index_group_handle = migs.group_handle
JOIN sys.dm_db_missing_index_details mid
    ON mig.index_handle = mid.index_handle
ORDER BY [Total Impact Score] DESC;
```
### Key DMV Columns Explained
| Column | Meaning |
| ----- | ----- |
|  | <p>Columns used in </p><p> conditions — should be </p><p>**key columns**</p> |
|  | <p>Columns used in </p><p>, </p><p>, </p><p> — should be </p><p>**last key columns**</p> |
|  | <p>Columns only in SELECT — should be </p><p>**INCLUDE columns**</p> |
|  | Estimated % query cost reduction if index is created |
|  | How many times this index would have been used |
### ⚠️ Important Caveats
- DMV data is **reset on SQL Server restart** — don't act on fresh restart data
- The optimizer suggests indexes **per query** — you may get duplicates or near-duplicates
- Don't blindly create every suggested index — evaluate overlap with existing indexes
- More indexes = slower writes (INSERT/UPDATE/DELETE must update every index)
- Always test the suggested index in a non-production environment first
---

## 7. Index Seek vs Index Scan vs Table Scan
These are the three ways SQL Server can retrieve data from a table, visible in **execution plans**.

### Table Scan
SQL Server reads **every single row** in the table from start to finish. This happens when there is no usable index, or the optimizer decides a scan is cheaper (e.g., returning most rows anyway).

```sql
-- No index on Status → Table Scan
SELECT * FROM Orders WHERE Status = 'Pending';
```
```
Cost:    O(n) — linear with table size
When:    No usable index exists, or returning > ~30% of rows
Icon:    📋 Table Scan (in execution plan)
```
### Index Scan
SQL Server reads **all (or most) pages of the index** from start to finish. It's like a table scan but on the index structure. This happens when the index exists but the query needs to read a large portion of it, or when columns not in the index force a full scan.

```sql
-- Index exists on CustomerID but query returns too many rows
-- → SQL Server may choose Index Scan over Seek + many Key Lookups
SELECT CustomerID, OrderDate FROM Orders;
```
```
Cost:    O(n) in the index — but index may be narrower than table
When:    Large range queries, non-selective filters, or SELECT *
Icon:    🔍 Index Scan (in execution plan)
```
### Index Seek
SQL Server navigates the **B-Tree structure** directly to find matching rows. This is the most efficient operation — it jumps straight to the relevant rows without reading the entire index.

```sql
-- Selective filter on indexed column → Index Seek
SELECT * FROM Orders WHERE OrderID = 5000;
SELECT * FROM Orders WHERE CustomerID = 101 AND OrderDate > '2024-01-01';
```
```
Cost:    O(log n) — logarithmic, extremely fast even on huge tables
When:    Selective WHERE clause on leading index columns
Icon:    🎯 Index Seek (in execution plan)
```
### Visual Comparison
```
Table Scan:
[Row1][Row2][Row3]...[Row999999]  ← reads everything

Index Scan:
[Idx1][Idx2][Idx3]...[Idx999999] ← reads all index entries

Index Seek (B-Tree Navigation):
        [Root]
       /      \
   [A-M]      [N-Z]
   /    \
[A-F]  [G-M]
         |
       [Result]  ← jumps directly here
```
### What Forces a Scan Instead of Seek?
```sql
-- ❌ Non-SARGable — function on column prevents seek
SELECT * FROM Orders WHERE YEAR(OrderDate) = 2024;

-- ✅ SARGable — seek is possible
SELECT * FROM Orders
WHERE OrderDate >= '2024-01-01' AND OrderDate < '2025-01-01';

-- ❌ Leading wildcard — can't seek
SELECT * FROM Customers WHERE LastName LIKE '%Smith';

-- ✅ Trailing wildcard — seek is possible
SELECT * FROM Customers WHERE LastName LIKE 'Smith%';

-- ❌ Implicit conversion — prevents seek
SELECT * FROM Orders WHERE CustomerID = '101'; -- CustomerID is INT
```
### Quick Decision Guide
| Scenario | Expected Operation | Performance |
| ----- | ----- | ----- |
|  | Index Seek | ⭐⭐⭐⭐⭐ |
|  | Index Seek | ⭐⭐⭐⭐⭐ |
|  | Index Seek (range) | ⭐⭐⭐⭐ |
|  | Index Scan | ⭐⭐⭐ |
|  | Table/Index Scan | ⭐⭐ |
| No usable index | Table Scan | ⭐ |
---

## 8. Columnstore Indexes
### What is a Columnstore Index?
Traditional indexes store data **row by row** (rowstore). Columnstore indexes store data **column by column**, compressing each column separately. This is revolutionary for analytical queries that aggregate a few columns across millions of rows.

```
Rowstore (Traditional):
Row1: [OrderID=1, CustomerID=101, Amount=500, Date=2024-01-01, Status='Active', ...]
Row2: [OrderID=2, CustomerID=102, Amount=750, Date=2024-01-02, Status='Pending', ...]

Columnstore:
OrderID column:    [1, 2, 3, 4, 5, ...]       ← stored together, highly compressed
CustomerID column: [101, 102, 101, 103, ...]   ← stored together
Amount column:     [500, 750, 200, 900, ...]   ← stored together
```
### Types of Columnstore Indexes
**Clustered Columnstore Index (CCI)** — The entire table is stored in columnar format. Ideal for data warehouse fact tables. Replaces the rowstore entirely.

```sql
CREATE CLUSTERED COLUMNSTORE INDEX CCI_FactSales
ON FactSales;
```
**Non-Clustered Columnstore Index (NCCI)** — A columnar index alongside a rowstore table. Good for mixed OLTP + analytics workloads (HTAP).

```sql
CREATE NONCLUSTERED COLUMNSTORE INDEX NCCI_Orders_Analytics
ON Orders (CustomerID, OrderDate, Amount, Status);
```
### Why Columnstore is Fast for Analytics
**Batch Mode Processing** — Columnstore enables batch mode execution, where SQL Server processes 900 rows at a time instead of one row at a time. This dramatically reduces CPU overhead.

**High Compression** — Columns of the same data type compress extremely well (often 5–10x). Less data to read from disk.

**Column Pruning** — If your query only touches 3 of 50 columns, only those 3 columns' data is read from disk. Rowstore must read the entire row.

```sql
-- This query reads ONLY the Amount and OrderDate columns
-- Columnstore skips all other columns entirely
SELECT
    YEAR(OrderDate) AS Year,
    SUM(Amount) AS TotalSales
FROM FactSales
GROUP BY YEAR(OrderDate);
```
### Columnstore vs Rowstore
| Feature | Rowstore Index | Columnstore Index |
| ----- | ----- | ----- |
| Best for | OLTP (point lookups, single rows) | Analytics (aggregations, large scans) |
| Compression | Low | Very high (5–10x) |
| Read performance (analytics) | Slow | Very fast |
| Write performance | Fast | Slower (delta store overhead) |
| Single row lookup | Fast (seek) | Slow |
| Aggregation queries | Slow | Very fast (batch mode) |
### Delta Store
New rows inserted into a columnstore index go into a **delta store** (a small rowstore buffer) first. When the delta store fills up (~1M rows), it gets compressed and moved into the columnstore as a new **row group**. This is handled automatically by a background process called the **tuple mover**.

```sql
-- Check row group health and delta store status
SELECT
    OBJECT_NAME(object_id)    AS TableName,
    row_group_id,
    state_description,
    total_rows,
    deleted_rows,
    size_in_bytes
FROM sys.column_store_row_groups
ORDER BY object_id, row_group_id;
```
### Best Use Cases
- Data warehouse fact tables with hundreds of millions of rows
- Historical reporting tables (read-heavy, rarely updated)
- Mixed workload tables where you want both fast OLTP and fast analytics (NCCI)
- ETL staging tables
---

## Index Design Cheat Sheet
```
Question to ask                          → Action
─────────────────────────────────────────────────────────────
Single row lookup by primary key?        → Clustered Index (default PK)
Frequently filter on a column?           → Non-Clustered Index on that column
Query returns extra columns (Key Lookup)?→ Add INCLUDE columns
Filter on a small subset of rows?        → Filtered Index
Table scans on a large analytics table?  → Columnstore Index
Index fragmentation > 30%?               → REBUILD
Index fragmentation 10-30%?              → REORGANIZE
Seeing slow queries?                     → Check Missing Index DMVs
Function in WHERE clause?                → Rewrite to be SARGable
```
---

## Summary Table
| Index Type | Best For | Key Limitation |
| ----- | ----- | ----- |
| Clustered | Primary key lookups, range scans | Only 1 per table |
| Non-Clustered | Secondary lookups, foreign keys | Key Lookup overhead |
| Composite | Multi-column filters | Column order matters |
| Covering | Eliminating Key Lookups | Can get wide/bloated |
| Filtered | Sparse columns, subset queries | Must match query predicate |
| Columnstore | Analytics, aggregations | Poor for single row lookups |




<!--- Eraser file: https://app.eraser.io/workspace/C05GfTfsKeIoc8NBflEl --->