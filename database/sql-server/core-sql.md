<p><a target="_blank" href="https://app.eraser.io/workspace/wIcLg18cn5ca0LqmzC44" id="edit-in-eraser-github-link"><img alt="Edit in Eraser" src="https://firebasestorage.googleapis.com/v0/b/second-petal-295822.appspot.com/o/images%2Fgithub%2FOpen%20in%20Eraser.svg?alt=media&amp;token=968381c8-a7e7-472a-8ed6-4a6626da5501"></a></p>


---

## Table of Contents

1. [﻿Joins](https://app.eraser.io/workspace/wIcLg18cn5ca0LqmzC44#1-joins)
2. [﻿Subqueries vs CTEs vs Temp Tables vs Table Variables](https://app.eraser.io/workspace/wIcLg18cn5ca0LqmzC44#2-subqueries-vs-ctes-vs-temp-tables-vs-table-variables)
3. [﻿Window Functions](https://app.eraser.io/workspace/wIcLg18cn5ca0LqmzC44#3-window-functions)
4. [﻿Aggregations — GROUP BY / HAVING](https://app.eraser.io/workspace/wIcLg18cn5ca0LqmzC44#4-aggregations--group-by--having)
5. [﻿UNION vs UNION ALL](https://app.eraser.io/workspace/wIcLg18cn5ca0LqmzC44#5-union-vs-union-all)
6. [﻿EXISTS vs IN vs JOIN](https://app.eraser.io/workspace/wIcLg18cn5ca0LqmzC44#6-exists-vs-in-vs-join)
7. [﻿CASE Expressions](https://app.eraser.io/workspace/wIcLg18cn5ca0LqmzC44#7-case-expressions)
8. [﻿PIVOT and UNPIVOT](https://app.eraser.io/workspace/wIcLg18cn5ca0LqmzC44#8-pivot-and-unpivot)
9. [﻿String, Date, and Math Functions](https://app.eraser.io/workspace/wIcLg18cn5ca0LqmzC44#9-string-date-and-math-functions)

---

## 1. Joins

### How Joins Work Internally

SQL Server uses three physical join operators under the hood. The optimizer picks one based on table size, indexes, and statistics:

| Operator | Best For | Requires Sort? | Memory |
|---|---|---|---|
| **Nested Loops** | Small outer, indexed inner | No | Low |
| **Merge Join** | Large sorted inputs | Yes (or pre-sorted) | Low |
| **Hash Join** | Large unsorted inputs | No | High (builds hash table) |

Understanding which operator is chosen — and why — is what separates senior devs from juniors.

---

### INNER JOIN

Returns only rows that have matching values in both tables.

```sql
SELECT o.OrderID, c.CustomerName
FROM Orders o
INNER JOIN Customers c ON o.CustomerID = c.CustomerID;
```

> **Senior note:** Always alias tables. Never rely on implicit column resolution — it breaks when schemas change and confuses the optimizer's statistics matching.

---

### LEFT JOIN (LEFT OUTER JOIN)

Returns all rows from the left table and matched rows from the right. Unmatched right-side rows return NULL.

```sql
SELECT c.CustomerName, o.OrderID
FROM Customers c
LEFT JOIN Orders o ON c.CustomerID = o.CustomerID;

-- Find customers with NO orders
SELECT c.CustomerName
FROM Customers c
LEFT JOIN Orders o ON c.CustomerID = o.CustomerID
WHERE o.OrderID IS NULL;
```

> **Senior note:** The `WHERE o.OrderID IS NULL` pattern is a classic anti-join. It's functionally equivalent to `NOT EXISTS` but can perform differently. The optimizer often rewrites them identically, but test with your actual data.

---

### RIGHT JOIN

Mirror of LEFT JOIN — rarely used in practice. Most developers rewrite RIGHT JOINs as LEFT JOINs by swapping table order for readability.

```sql
-- These two are identical in result
SELECT * FROM A RIGHT JOIN B ON A.id = B.id;
SELECT * FROM B LEFT JOIN A ON B.id = A.id;  -- preferred
```

---

### FULL OUTER JOIN

Returns all rows from both tables. NULLs fill in where there's no match on either side.

```sql
SELECT c.CustomerName, o.OrderID
FROM Customers c
FULL OUTER JOIN Orders o ON c.CustomerID = o.CustomerID;
```

> **Senior note:** FULL OUTER JOINs are often a sign of a data quality problem — they're typically used to find mismatches between two systems or to reconcile data.

---

### CROSS JOIN

Produces a Cartesian product — every row from the left paired with every row from the right. No ON clause.

```sql
-- Generate all date/product combinations for a sales matrix
SELECT d.DateValue, p.ProductName
FROM DimDate d
CROSS JOIN Products p;
```

> **Senior note:** CROSS JOINs are deliberate and useful for generating combinations, calendar scaffolding, or test data. If you see one in production code without a comment explaining why, it's probably a bug.

---

### SELF JOIN

A table joined to itself. Used for hierarchical or comparative queries.

```sql
-- Find employees and their managers (same table)
SELECT e.EmployeeName, m.EmployeeName AS ManagerName
FROM Employees e
LEFT JOIN Employees m ON e.ManagerID = m.EmployeeID;

-- Find pairs of products in the same category
SELECT a.ProductName, b.ProductName
FROM Products a
JOIN Products b ON a.CategoryID = b.CategoryID
AND a.ProductID < b.ProductID;  -- avoid duplicate pairs
```

---

### Join Performance Tips (Senior Level)

**Filter before joining.** Push WHERE conditions into a subquery or CTE before joining large tables.

```sql
-- BAD: joins all orders then filters
SELECT c.CustomerName, o.OrderID
FROM Customers c
JOIN Orders o ON c.CustomerID = o.CustomerID
WHERE o.OrderDate > '2024-01-01';

-- BETTER: when Orders is huge, pre-filter first
SELECT c.CustomerName, o.OrderID
FROM Customers c
JOIN (
    SELECT OrderID, CustomerID
    FROM Orders
    WHERE OrderDate > '2024-01-01'
) o ON c.CustomerID = o.CustomerID;
```

**Join on indexed columns.** The join predicate column should have an index on at least one side — ideally both.

**Watch for implicit type conversions.** Joining `INT` to `VARCHAR` causes an implicit cast that makes indexes non-SARGable.

```sql
-- BAD: CustomerID is INT but joined to VARCHAR — index on CustomerID is NOT used
JOIN Customers c ON c.CustomerID = o.CustomerID_Varchar

-- FIX: cast explicitly and consistently
JOIN Customers c ON c.CustomerID = CAST(o.CustomerID_Varchar AS INT)
```

---

## 2. Subqueries vs CTEs vs Temp Tables vs Table Variables

This is one of the most nuanced topics in SQL Server. Each has distinct performance characteristics that matter at scale.

---

### Subqueries

A query nested inside another query. Can appear in SELECT, FROM, or WHERE clauses.

```sql
-- Correlated subquery (executes once per outer row — can be slow)
SELECT CustomerName,
    (SELECT COUNT(*) FROM Orders o WHERE o.CustomerID = c.CustomerID) AS OrderCount
FROM Customers c;

-- Non-correlated subquery (executes once)
SELECT * FROM Orders
WHERE CustomerID IN (
    SELECT CustomerID FROM Customers WHERE Country = 'USA'
);
```

> **Senior note:** Correlated subqueries are dangerous at scale. They execute once per row of the outer query. A correlated subquery against a million-row table can execute a million times. Rewrite as a JOIN or window function wherever possible.

---

### CTEs (Common Table Expressions)

Named result sets defined with `WITH`. They improve readability but are **not materialized** — they're inlined into the query each time they're referenced.

```sql
WITH RecentOrders AS (
    SELECT CustomerID, COUNT(*) AS OrderCount
    FROM Orders
    WHERE OrderDate >= DATEADD(YEAR, -1, GETDATE())
    GROUP BY CustomerID
),
HighValueCustomers AS (
    SELECT CustomerID
    FROM Customers
    WHERE CustomerTier = 'Gold'
)
SELECT c.CustomerName, r.OrderCount
FROM Customers c
JOIN RecentOrders r ON c.CustomerID = r.CustomerID
JOIN HighValueCustomers h ON c.CustomerID = h.CustomerID;
```

> **Senior note:** A CTE referenced **twice** in the same query is executed **twice** — it is NOT cached. This is a common misconception. If you need the result more than once, materialize it into a temp table instead.

```sql
-- CTE executed twice — expensive if the CTE is complex
WITH ExpensiveCTE AS (SELECT ... FROM HugeTable)
SELECT * FROM ExpensiveCTE e1
JOIN ExpensiveCTE e2 ON e1.id = e2.parent_id;

-- Better: materialize once
SELECT ... INTO #temp FROM HugeTable;
SELECT * FROM #temp t1 JOIN #temp t2 ON t1.id = t2.parent_id;
```

---

### Recursive CTEs

CTEs can reference themselves for hierarchical traversal.

```sql
WITH OrgHierarchy AS (
    -- Anchor: start with top-level managers
    SELECT EmployeeID, ManagerID, EmployeeName, 0 AS Level
    FROM Employees
    WHERE ManagerID IS NULL

    UNION ALL

    -- Recursive: join each employee to their manager
    SELECT e.EmployeeID, e.ManagerID, e.EmployeeName, h.Level + 1
    FROM Employees e
    JOIN OrgHierarchy h ON e.ManagerID = h.EmployeeID
)
SELECT * FROM OrgHierarchy
ORDER BY Level, EmployeeName;
```

> **Senior note:** Always add `OPTION (MAXRECURSION n)` to recursive CTEs. The default is 100 levels. Circular references will cause infinite recursion without a termination check.

```sql
-- Allow up to 500 levels; 0 = unlimited (dangerous)
SELECT * FROM OrgHierarchy
OPTION (MAXRECURSION 500);
```

---

### Temp Tables (#temp)

Physically created in **TempDB**. They have real statistics, can be indexed, and persist for the session duration.

```sql
-- Create and populate
SELECT CustomerID, SUM(Amount) AS TotalSpend
INTO #CustomerSpend
FROM Orders
GROUP BY CustomerID;

-- Add index after creation
CREATE INDEX IX_CustomerSpend_CustomerID ON #CustomerSpend (CustomerID);

-- Use it
SELECT c.CustomerName, s.TotalSpend
FROM Customers c
JOIN #CustomerSpend s ON c.CustomerID = s.CustomerID;

-- Clean up (good practice, though dropped automatically at session end)
DROP TABLE IF EXISTS #CustomerSpend;
```

> **Senior note:** Temp tables are the right choice when: (a) you need to reference the result multiple times, (b) the result set is large, (c) you need to add indexes on intermediate results, or (d) you need the optimizer to build accurate statistics for subsequent queries.

---

### Table Variables (@table)

Declared in memory (though they can spill to TempDB). No statistics, no indexes (except a primary key), limited scope.

```sql
DECLARE @CustomerIDs TABLE (CustomerID INT PRIMARY KEY);

INSERT INTO @CustomerIDs
SELECT CustomerID FROM Customers WHERE Country = 'USA';

SELECT o.*
FROM Orders o
JOIN @CustomerIDs t ON o.CustomerID = t.CustomerID;
```

> **Senior note:** Table variables have **no statistics**. The optimizer always estimates 1 row regardless of actual row count. This leads to bad plan choices (nested loops when hash join would be better) on large datasets. Use temp tables for anything over a few hundred rows.

---

### When to Use What — Decision Guide

```
Is the result used only once?
    YES → CTE or subquery (simpler, cleaner)
    NO  → Temp table (materialized, indexed)

Is the dataset large (> 1000 rows)?
    YES → Temp table (optimizer gets real stats)
    NO  → Table variable or CTE is fine

Do you need indexes on the intermediate result?
    YES → Temp table (only option)
    NO  → Any of the above

Is this inside a loop or called frequently?
    YES → Temp table (avoid repeated CTE re-execution)
    NO  → CTE is fine
```

---

## 3. Window Functions

Window functions perform calculations across a "window" of rows related to the current row — without collapsing the result set like GROUP BY does. They are one of the most powerful features in modern SQL.

```sql
-- Basic syntax
function_name() OVER (
    PARTITION BY column1       -- divide rows into groups
    ORDER BY column2           -- define row order within partition
    ROWS BETWEEN ... AND ...   -- optional frame clause
)
```

---

### ROW_NUMBER()

Assigns a unique sequential number to each row within a partition.

```sql
-- Rank orders per customer by date
SELECT
    CustomerID, OrderID, OrderDate,
    ROW_NUMBER() OVER (PARTITION BY CustomerID ORDER BY OrderDate DESC) AS RowNum
FROM Orders;

-- Get the most recent order per customer
SELECT * FROM (
    SELECT *,
        ROW_NUMBER() OVER (PARTITION BY CustomerID ORDER BY OrderDate DESC) AS rn
    FROM Orders
) x
WHERE rn = 1;
```

> **Senior note:** ROW_NUMBER() for deduplication is one of the most common production patterns. It always assigns unique numbers even for ties.

---

### RANK() and DENSE_RANK()

```sql
SELECT
    ProductID, SalesAmount,
    RANK()       OVER (ORDER BY SalesAmount DESC) AS Rank,
    DENSE_RANK() OVER (ORDER BY SalesAmount DESC) AS DenseRank
FROM Sales;
```

| SalesAmount | RANK | DENSE_RANK |
|---|---|---|
| 1000 | 1 | 1 |
| 900 | 2 | 2 |
| 900 | 2 | 2 |
| 800 | 4 | 3 |

**RANK** skips numbers after ties (1, 2, 2, 4). **DENSE_RANK** does not skip (1, 2, 2, 3).

---

### LEAD() and LAG()

Access values from the next or previous row without a self-join.

```sql
SELECT
    OrderDate,
    Amount,
    LAG(Amount, 1, 0)  OVER (PARTITION BY CustomerID ORDER BY OrderDate) AS PrevAmount,
    LEAD(Amount, 1, 0) OVER (PARTITION BY CustomerID ORDER BY OrderDate) AS NextAmount,
    Amount - LAG(Amount, 1, 0) OVER (PARTITION BY CustomerID ORDER BY OrderDate) AS Change
FROM Orders;
```

> **Senior note:** Before LAG/LEAD existed, this required a self-join with a row number — much more expensive. Always use LAG/LEAD for this pattern.

---

### Running Totals and Moving Averages

```sql
-- Running total (cumulative sum)
SELECT
    OrderDate, Amount,
    SUM(Amount) OVER (
        PARTITION BY CustomerID
        ORDER BY OrderDate
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS RunningTotal,

    -- 7-day moving average
    AVG(Amount) OVER (
        PARTITION BY CustomerID
        ORDER BY OrderDate
        ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
    ) AS MovingAvg7Day
FROM Orders;
```

### Frame Clause Options

```
ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW  -- running total
ROWS BETWEEN 6 PRECEDING AND CURRENT ROW          -- rolling 7 rows
ROWS BETWEEN CURRENT ROW AND UNBOUNDED FOLLOWING  -- reverse running total
ROWS BETWEEN 3 PRECEDING AND 3 FOLLOWING          -- centered window
RANGE BETWEEN ...                                 -- value-based (not row-based)
```

> **Senior note:** `ROWS` is almost always preferable to `RANGE`. `RANGE` uses value-based comparison and can be significantly slower because it can't use a fixed-size sliding window.

---

### NTILE()

Divides rows into N roughly equal buckets.

```sql
-- Divide customers into 4 quartiles by total spend
SELECT
    CustomerID, TotalSpend,
    NTILE(4) OVER (ORDER BY TotalSpend DESC) AS SpendQuartile
FROM CustomerSummary;
```

---

## 4. Aggregations — GROUP BY / HAVING

### The Logical Order of Execution

Understanding query execution order is critical. SQL Server processes clauses in this logical order (not the written order):

```
1. FROM / JOIN     — identify source tables
2. WHERE           — filter rows
3. GROUP BY        — group remaining rows
4. HAVING          — filter groups
5. SELECT          — compute output columns
6. DISTINCT        — remove duplicates
7. ORDER BY        — sort
8. TOP / OFFSET    — limit rows
```

> **Senior note:** This is why you can't use a SELECT alias in a WHERE clause — WHERE is processed before SELECT. But you CAN use a SELECT alias in ORDER BY.

```sql
-- ERROR: alias not available in WHERE
SELECT SalesAmount * 0.1 AS Tax
FROM Sales
WHERE Tax > 100;  -- ❌ 'Tax' is unknown

-- OK: alias available in ORDER BY
SELECT SalesAmount * 0.1 AS Tax
FROM Sales
ORDER BY Tax DESC;  -- ✅

-- FIX for WHERE: repeat the expression or use a subquery
SELECT * FROM (
    SELECT SalesAmount * 0.1 AS Tax FROM Sales
) x
WHERE Tax > 100;
```

---

### GROUP BY Internals

```sql
SELECT CustomerID, COUNT(*) AS OrderCount, SUM(Amount) AS Total
FROM Orders
WHERE OrderDate >= '2024-01-01'
GROUP BY CustomerID
HAVING COUNT(*) > 5
ORDER BY Total DESC;
```

**HAVING vs WHERE:** WHERE filters rows before grouping; HAVING filters groups after aggregation. Never use HAVING where WHERE would suffice — it forces the engine to build the group before discarding it.

```sql
-- BAD: builds all groups then discards non-2024 ones
SELECT CustomerID, COUNT(*)
FROM Orders
GROUP BY CustomerID
HAVING MIN(OrderDate) >= '2024-01-01';

-- BETTER: filter first, group fewer rows
SELECT CustomerID, COUNT(*)
FROM Orders
WHERE OrderDate >= '2024-01-01'
GROUP BY CustomerID;
```

---

### GROUPING SETS, ROLLUP, CUBE

Advanced aggregation that computes multiple GROUP BY levels in one pass.

```sql
-- ROLLUP: hierarchical subtotals
SELECT Year, Quarter, SUM(Sales)
FROM SalesFact
GROUP BY ROLLUP(Year, Quarter);
-- Produces: (Year, Quarter), (Year, NULL=subtotal), (NULL, NULL=grand total)

-- CUBE: all combinations
SELECT Region, ProductCategory, SUM(Sales)
FROM SalesFact
GROUP BY CUBE(Region, ProductCategory);
-- Produces all combinations: (R,P), (R,NULL), (NULL,P), (NULL,NULL)

-- GROUPING SETS: explicit combinations
SELECT Region, ProductCategory, SUM(Sales)
FROM SalesFact
GROUP BY GROUPING SETS (
    (Region, ProductCategory),
    (Region),
    ()
);
```

> **Senior note:** ROLLUP/CUBE are far more efficient than UNION ALL of multiple GROUP BYs — they scan the data once. A common mistake is writing 3 separate SELECT+GROUP BY queries unioned together when ROLLUP does it in one pass.

---

## 5. UNION vs UNION ALL

### The Difference

```sql
-- UNION ALL: keeps ALL rows including duplicates (faster)
SELECT CustomerID FROM Orders_2023
UNION ALL
SELECT CustomerID FROM Orders_2024;

-- UNION: removes duplicates (sorts/hashes to deduplicate — slower)
SELECT CustomerID FROM Orders_2023
UNION
SELECT CustomerID FROM Orders_2024;
```

> **Senior note:** UNION performs a sort or hash operation to eliminate duplicates — similar cost to DISTINCT. Unless you genuinely need deduplication, always use UNION ALL. Using UNION "just to be safe" is a common performance mistake.

---

### Rules for UNION

- Both queries must return the **same number of columns**
- Corresponding columns must be **compatible data types**
- Column names come from the **first query**
- ORDER BY applies to the **entire result**, not individual queries

```sql
-- Only one ORDER BY, at the end
SELECT ProductName, Price FROM Products WHERE CategoryID = 1
UNION ALL
SELECT ProductName, Price FROM Products WHERE CategoryID = 2
ORDER BY Price DESC;  -- applies to combined result
```

---

### Performance Consideration

```sql
-- BAD: two scans of the same table
SELECT * FROM Orders WHERE Status = 'Pending'
UNION ALL
SELECT * FROM Orders WHERE Status = 'Processing';

-- BETTER: one scan with OR (or IN)
SELECT * FROM Orders WHERE Status IN ('Pending', 'Processing');
```

---

## 6. EXISTS vs IN vs JOIN

This is a classic senior interview topic because the three approaches look similar but behave differently.

---

### IN with a Subquery

```sql
SELECT * FROM Orders
WHERE CustomerID IN (
    SELECT CustomerID FROM Customers WHERE Country = 'USA'
);
```

The subquery is evaluated and the full list is built first. If the subquery returns NULLs, `IN` comparisons against NULL silently fail — this is a subtle bug.

```sql
-- BUG: if subquery returns any NULL, nothing matches NOT IN
SELECT * FROM Orders
WHERE CustomerID NOT IN (
    SELECT CustomerID FROM Customers WHERE Country = 'USA'
    -- if any CustomerID is NULL, the entire NOT IN returns empty!
);

-- SAFE version: exclude NULLs explicitly
SELECT * FROM Orders
WHERE CustomerID NOT IN (
    SELECT CustomerID FROM Customers
    WHERE Country = 'USA' AND CustomerID IS NOT NULL
);
```

> **Senior note:** `NOT IN` with a nullable subquery is one of the most common subtle bugs in SQL. Always use `NOT EXISTS` instead of `NOT IN` when NULLs might be present.

---

### EXISTS

Short-circuits on the first match. More efficient when you only care about existence, not the actual values.

```sql
-- Does this customer have any orders?
SELECT c.CustomerName
FROM Customers c
WHERE EXISTS (
    SELECT 1  -- the SELECT list doesn't matter for EXISTS
    FROM Orders o
    WHERE o.CustomerID = c.CustomerID
);

-- Safe NOT EXISTS — not affected by NULLs
SELECT c.CustomerName
FROM Customers c
WHERE NOT EXISTS (
    SELECT 1 FROM Orders o
    WHERE o.CustomerID = c.CustomerID
);
```

> **Senior note:** `SELECT 1`, `SELECT *`, `SELECT NULL` inside EXISTS are all identical — the optimizer ignores the select list. But `SELECT 1` is conventional because it makes intent clear: we only care about existence.

---

### JOIN for Existence Checks

Using a JOIN to check existence introduces duplicates if the right table has multiple matching rows.

```sql
-- WRONG: returns duplicate customers if they have multiple orders
SELECT c.CustomerName
FROM Customers c
JOIN Orders o ON c.CustomerID = o.CustomerID;

-- FIX: use DISTINCT (but adds sort overhead)
SELECT DISTINCT c.CustomerName
FROM Customers c
JOIN Orders o ON c.CustomerID = o.CustomerID;

-- BETTER: use EXISTS
SELECT c.CustomerName
FROM Customers c
WHERE EXISTS (SELECT 1 FROM Orders o WHERE o.CustomerID = c.CustomerID);
```

---

### Performance Summary

| Approach | Best Use Case | NULL Safe? |
|---|---|---|
| `IN` | Small, static lists or non-nullable subqueries | No (NOT IN) |
| `EXISTS` | Large tables, correlated checks, any nullable column | Yes |
| `JOIN` | Need columns from both tables | Causes duplicates |

> **Senior note:** In modern SQL Server (2005+), the optimizer often rewrites IN and EXISTS to the same execution plan. But EXISTS is semantically clearer and safer — prefer it for correlated checks.

---

## 7. CASE Expressions

### Simple vs Searched CASE

```sql
-- Simple CASE: compares one expression to multiple values
SELECT OrderID,
    CASE Status
        WHEN 'P' THEN 'Pending'
        WHEN 'C' THEN 'Completed'
        WHEN 'X' THEN 'Cancelled'
        ELSE 'Unknown'
    END AS StatusDescription
FROM Orders;

-- Searched CASE: evaluates separate boolean conditions
SELECT OrderID, Amount,
    CASE
        WHEN Amount >= 10000 THEN 'Platinum'
        WHEN Amount >= 5000  THEN 'Gold'
        WHEN Amount >= 1000  THEN 'Silver'
        ELSE 'Standard'
    END AS Tier
FROM Orders;
```

---

### CASE in Aggregations (Conditional Aggregation)

This pattern replaces multiple queries or PIVOTs with a single scan.

```sql
-- Count orders by status in one pass (instead of multiple queries)
SELECT
    CustomerID,
    COUNT(*)                                          AS TotalOrders,
    SUM(CASE WHEN Status = 'Completed' THEN 1 ELSE 0 END) AS Completed,
    SUM(CASE WHEN Status = 'Pending'   THEN 1 ELSE 0 END) AS Pending,
    SUM(CASE WHEN Status = 'Cancelled' THEN 1 ELSE 0 END) AS Cancelled,
    SUM(CASE WHEN Status = 'Completed' THEN Amount ELSE 0 END) AS CompletedRevenue
FROM Orders
GROUP BY CustomerID;
```

> **Senior note:** Conditional aggregation is more efficient than multiple GROUP BY queries unioned together. It scans the table once and pivots in memory. This is the preferred approach for cross-tab style reports.

---

### CASE in ORDER BY

```sql
-- Prioritize 'Pending' first, then others alphabetically
SELECT * FROM Orders
ORDER BY
    CASE WHEN Status = 'Pending' THEN 0 ELSE 1 END,
    Status;
```

---

### IIF() and CHOOSE() — Shorthand

```sql
-- IIF: shorthand for 2-branch CASE
SELECT IIF(Amount > 1000, 'High', 'Low') AS AmountBand FROM Orders;

-- CHOOSE: pick from a list by index (1-based)
SELECT CHOOSE(MONTH(OrderDate), 'Q1','Q1','Q1','Q2','Q2','Q2',
                                 'Q3','Q3','Q3','Q4','Q4','Q4') AS Quarter
FROM Orders;
```

---

## 8. PIVOT and UNPIVOT

### PIVOT — Rows to Columns

```sql
-- Sales by product category across years
SELECT *
FROM (
    SELECT Year, Category, Amount FROM Sales
) src
PIVOT (
    SUM(Amount)
    FOR Category IN ([Electronics], [Clothing], [Food])
) pvt;
```

Output:
```
Year  | Electronics | Clothing | Food
2022  | 50000       | 30000    | 20000
2023  | 65000       | 35000    | 22000
```

> **Senior note:** PIVOT requires you to hard-code the column values. For dynamic column lists, you need dynamic SQL to build the IN list at runtime.

---

### Dynamic PIVOT

```sql
DECLARE @cols NVARCHAR(MAX);
DECLARE @sql  NVARCHAR(MAX);

-- Build column list dynamically
SELECT @cols = STRING_AGG(QUOTENAME(Category), ',')
FROM (SELECT DISTINCT Category FROM Sales) c;

SET @sql = N'
    SELECT * FROM (
        SELECT Year, Category, Amount FROM Sales
    ) src
    PIVOT (
        SUM(Amount) FOR Category IN (' + @cols + ')
    ) pvt;';

EXEC sp_executesql @sql;
```

---

### UNPIVOT — Columns to Rows

```sql
-- Turn wide table back into narrow format
SELECT Year, Category, Amount
FROM SalesWide
UNPIVOT (
    Amount FOR Category IN (Electronics, Clothing, Food)
) unpvt;
```

> **Senior note:** UNPIVOT is essentially syntactic sugar for UNION ALL across columns. For simple cases with known columns it's clean. For complex transformations, CROSS APPLY with VALUES is more flexible.

```sql
-- CROSS APPLY VALUES alternative (more flexible than UNPIVOT)
SELECT s.Year, v.Category, v.Amount
FROM SalesWide s
CROSS APPLY (VALUES
    ('Electronics', s.Electronics),
    ('Clothing',    s.Clothing),
    ('Food',        s.Food)
) v(Category, Amount);
```

---

## 9. String, Date, and Math Functions

### String Functions — Senior-Level Patterns

```sql
-- STRING_AGG: concatenate values (SQL Server 2017+)
SELECT CustomerID,
    STRING_AGG(ProductName, ', ') WITHIN GROUP (ORDER BY ProductName) AS Products
FROM OrderItems
GROUP BY CustomerID;

-- STRING_SPLIT: split delimited string into rows
SELECT value AS Tag
FROM STRING_SPLIT('sql,server,performance', ',');

-- STUFF + FOR XML PATH: pre-2017 string aggregation
SELECT CustomerID,
    STUFF((
        SELECT ', ' + ProductName
        FROM OrderItems oi
        WHERE oi.CustomerID = o.CustomerID
        FOR XML PATH(''), TYPE
    ).value('.', 'NVARCHAR(MAX)'), 1, 2, '') AS Products
FROM Orders o
GROUP BY CustomerID;

-- Pattern matching
SELECT * FROM Customers WHERE Email LIKE '%@gmail.com';        -- ends with
SELECT * FROM Products WHERE ProductCode LIKE '[A-Z][0-9][0-9]%'; -- pattern
SELECT * FROM Products WHERE ProductCode LIKE '[^0-9]%';       -- NOT starting with digit

-- CHARINDEX vs PATINDEX
SELECT CHARINDEX('sql', 'learn sql server');   -- returns 7 (position)
SELECT PATINDEX('%[0-9]%', 'abc123def');       -- returns 4 (supports patterns)

-- Trimming (SQL Server 2017+)
SELECT TRIM('  hello  ');          -- both sides
SELECT TRIM('x' FROM 'xxxhelloxxx'); -- trim specific character

-- FORMAT (locale-aware, but SLOW — avoid in large queries)
SELECT FORMAT(1234567.89, 'N2', 'en-US');  -- 1,234,567.89
-- Better for performance:
SELECT CONVERT(VARCHAR, CAST(1234567.89 AS MONEY), 1);  -- faster
```

---

### Date Functions — Senior-Level Patterns

```sql
-- Core functions
SELECT GETDATE();           -- current datetime (local)
SELECT GETUTCDATE();        -- current UTC datetime
SELECT SYSDATETIMEOFFSET(); -- datetime with timezone offset

-- DATEADD / DATEDIFF
SELECT DATEADD(DAY, 30, GETDATE());           -- 30 days from now
SELECT DATEADD(MONTH, -3, GETDATE());         -- 3 months ago
SELECT DATEDIFF(DAY, '2024-01-01', GETDATE()); -- days between

-- DATEDIFF traps — it counts boundaries, not durations
SELECT DATEDIFF(YEAR, '2024-12-31', '2025-01-01'); -- returns 1, not 0!

-- Date truncation (SQL Server has no TRUNC — use DATEADD/DATEDIFF)
SELECT DATEADD(DAY,   DATEDIFF(DAY,   0, GETDATE()), 0); -- start of today
SELECT DATEADD(MONTH, DATEDIFF(MONTH, 0, GETDATE()), 0); -- start of month
SELECT DATEADD(YEAR,  DATEDIFF(YEAR,  0, GETDATE()), 0); -- start of year

-- SQL Server 2022+ simplified
SELECT DATETRUNC(MONTH, GETDATE());  -- start of current month

-- EOMONTH: last day of month
SELECT EOMONTH(GETDATE());          -- last day of current month
SELECT EOMONTH(GETDATE(), 1);       -- last day of next month

-- Date parts
SELECT YEAR(GETDATE()), MONTH(GETDATE()), DAY(GETDATE());
SELECT DATEPART(WEEKDAY, GETDATE()); -- 1=Sunday by default (@@DATEFIRST dependent!)
SELECT DATENAME(MONTH, GETDATE());  -- 'February'

-- Safe string to date conversion
SELECT TRY_CONVERT(DATE, '2024-13-01'); -- returns NULL instead of error
SELECT TRY_CAST('not-a-date' AS DATE);  -- returns NULL instead of error

-- SARGable date range filter (index-friendly)
-- BAD: function on column — index not used
WHERE YEAR(OrderDate) = 2024

-- GOOD: range comparison — index is used
WHERE OrderDate >= '2024-01-01' AND OrderDate < '2025-01-01'
```

---

### Math Functions — Senior-Level Patterns

```sql
-- Rounding behavior — know the differences
SELECT ROUND(2.345, 2);    -- 2.350 (rounds up)
SELECT ROUND(2.355, 2);    -- 2.360
SELECT FLOOR(2.9);         -- 2 (always down)
SELECT CEILING(2.1);       -- 3 (always up)
SELECT TRUNCATE(2.9, 0);   -- not in SQL Server; use ROUND(x, 0, 1)
SELECT ROUND(2.9, 0, 1);   -- 2 (truncate, not round)

-- Integer division vs float division
SELECT 7 / 2;              -- 3 (integer division — common bug!)
SELECT 7 / 2.0;            -- 3.5
SELECT CAST(7 AS FLOAT) / 2; -- 3.5

-- Handling division by zero
SELECT Amount / NULLIF(Quantity, 0) AS UnitPrice  -- returns NULL if Quantity = 0
FROM OrderItems;

-- Modulo
SELECT 17 % 5;  -- 2

-- ABS, POWER, SQRT, LOG
SELECT ABS(-42);           -- 42
SELECT POWER(2.0, 10);     -- 1024
SELECT SQRT(144);          -- 12
SELECT LOG(100, 10);       -- 2 (log base 10)
SELECT LOG(EXP(1));        -- 1 (natural log)
```

---

## Common Senior Interview Patterns

### Find Nth Highest Value

```sql
-- Using ROW_NUMBER
SELECT Salary FROM (
    SELECT Salary,
        DENSE_RANK() OVER (ORDER BY Salary DESC) AS Rnk
    FROM Employees
) x
WHERE Rnk = 3;  -- 3rd highest unique salary
```

### Delete Duplicates, Keep One

```sql
WITH Dupes AS (
    SELECT *,
        ROW_NUMBER() OVER (
            PARTITION BY CustomerID, OrderDate
            ORDER BY OrderID
        ) AS rn
    FROM Orders
)
DELETE FROM Dupes WHERE rn > 1;
```

### Running Total with Reset

```sql
-- Running total that resets when value goes negative
SELECT
    Date, Amount,
    SUM(Amount) OVER (
        ORDER BY Date
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS RunningTotal
FROM Transactions;
```

### Gap and Island Problems

```sql
-- Find consecutive date ranges (islands)
WITH Numbered AS (
    SELECT Date,
        ROW_NUMBER() OVER (ORDER BY Date) AS rn,
        DATEADD(DAY, -ROW_NUMBER() OVER (ORDER BY Date), Date) AS grp
    FROM ActiveDates
)
SELECT MIN(Date) AS StartDate, MAX(Date) AS EndDate, COUNT(*) AS Days
FROM Numbered
GROUP BY grp
ORDER BY StartDate;
```

---

## Quick Reference — Function Cheat Sheet

| Category | Function | Notes |
|---|---|---|
| String | `STRING_AGG` | Concat with delimiter (2017+) |
| String | `STRING_SPLIT` | Split to rows (2016+) |
| String | `TRIM/LTRIM/RTRIM` | Whitespace removal |
| String | `CHARINDEX` | Position of substring |
| String | `PATINDEX` | Position with pattern |
| String | `REPLACE` | Replace substring |
| String | `FORMAT` | Locale formatting (slow) |
| Date | `DATEADD/DATEDIFF` | Date arithmetic |
| Date | `EOMONTH` | End of month |
| Date | `DATETRUNC` | Truncate to unit (2022+) |
| Date | `TRY_CONVERT` | Safe type conversion |
| Math | `ROUND/FLOOR/CEILING` | Rounding variants |
| Math | `NULLIF` | Division by zero safety |
| Math | `ABS/POWER/SQRT` | Standard math |
| Window | `ROW_NUMBER` | Unique sequential rank |
| Window | `RANK/DENSE_RANK` | Rank with ties |
| Window | `LEAD/LAG` | Adjacent row access |
| Window | `SUM/AVG OVER` | Running aggregates |



<!--- Eraser file: https://app.eraser.io/workspace/wIcLg18cn5ca0LqmzC44 --->