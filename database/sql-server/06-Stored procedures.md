<p><a target="_blank" href="https://app.eraser.io/workspace/kAdixbBGpFADV91AzplZ" id="edit-in-eraser-github-link"><img alt="Edit in Eraser" src="https://firebasestorage.googleapis.com/v0/b/second-petal-295822.appspot.com/o/images%2Fgithub%2FOpen%20in%20Eraser.svg?alt=media&amp;token=968381c8-a7e7-472a-8ed6-4a6626da5501"></a></p>

# SQL Server Stored Procedures & Programmability
## Overview
SQL Server's programmability features allow you to encapsulate business logic directly in the database — close to the data, secure, compiled, and reusable. Understanding when and how to use each feature is critical for writing maintainable, high-performance T-SQL code.

```
Programmability Stack
─────────────────────────────────────────────────
Stored Procedures  ← Business logic, transactions
Functions (TVF)    ← Reusable table expressions
Dynamic SQL        ← Runtime-generated queries
Error Handling     ← Graceful failure management
Cursors            ← Row-by-row (avoid when possible)
Temp Tables / CTEs ← Intermediate result management
MERGE              ← Upsert operations
─────────────────────────────────────────────────
```
---

## 1. Input / Output Parameters
Parameters are how stored procedures **communicate with the outside world** — receiving data from callers and returning results back.

### Input Parameters
Input parameters pass values **into** a stored procedure. They are read-only inside the procedure.

```sql
-- Basic input parameters with data types and defaults
CREATE PROCEDURE usp_GetOrders
    @CustomerID INT,
    @FromDate   DATETIME = NULL,      -- optional, defaults to NULL
    @ToDate     DATETIME = NULL,      -- optional, defaults to NULL
    @Status     VARCHAR(20) = 'All'   -- optional, defaults to 'All'
AS
BEGIN
    SET NOCOUNT ON;  -- suppress "X rows affected" messages

    SELECT
        o.OrderID,
        o.OrderDate,
        o.TotalAmount,
        o.Status
    FROM Orders o
    WHERE o.CustomerID = @CustomerID
      AND (@FromDate IS NULL OR o.OrderDate >= @FromDate)
      AND (@ToDate   IS NULL OR o.OrderDate <= @ToDate)
      AND (@Status = 'All'   OR o.Status = @Status)
    ORDER BY o.OrderDate DESC;
END;

-- Calling with positional parameters
EXEC usp_GetOrders 101, '2024-01-01', '2024-12-31';

-- Calling with named parameters (recommended — order independent)
EXEC usp_GetOrders
    @CustomerID = 101,
    @FromDate   = '2024-01-01',
    @Status     = 'Pending';
    -- @ToDate uses its default (NULL)
```
### Output Parameters
Output parameters **return values back** to the caller. They must be declared with the `OUTPUT` keyword in both the procedure definition and the call.

```sql
CREATE PROCEDURE usp_CreateOrder
    @CustomerID  INT,
    @TotalAmount DECIMAL(18,2),
    @OrderID     INT     OUTPUT,   -- returns the new OrderID
    @StatusMsg   VARCHAR(200) OUTPUT  -- returns status message
AS
BEGIN
    SET NOCOUNT ON;

    BEGIN TRANSACTION;
    BEGIN TRY
        INSERT INTO Orders (CustomerID, TotalAmount, OrderDate, Status)
        VALUES (@CustomerID, @TotalAmount, GETDATE(), 'Pending');

        SET @OrderID   = SCOPE_IDENTITY();  -- capture generated ID
        SET @StatusMsg = 'Order created successfully';

        COMMIT TRANSACTION;
    END TRY
    BEGIN CATCH
        ROLLBACK TRANSACTION;
        SET @OrderID   = -1;
        SET @StatusMsg = 'Error: ' + ERROR_MESSAGE();
    END CATCH;
END;

-- Calling with output parameters
DECLARE @NewOrderID  INT;
DECLARE @Message     VARCHAR(200);

EXEC usp_CreateOrder
    @CustomerID  = 101,
    @TotalAmount = 599.99,
    @OrderID     = @NewOrderID  OUTPUT,
    @StatusMsg   = @Message     OUTPUT;

SELECT @NewOrderID AS NewOrderID, @Message AS Message;
```
### Return Values
Stored procedures can also return an integer **return code** via `RETURN`. By convention, 0 = success, non-zero = error. This is separate from output parameters.

```sql
CREATE PROCEDURE usp_ValidateCustomer
    @CustomerID INT
AS
BEGIN
    IF NOT EXISTS (SELECT 1 FROM Customers WHERE CustomerID = @CustomerID)
        RETURN -1;  -- customer not found

    IF EXISTS (SELECT 1 FROM Customers
               WHERE CustomerID = @CustomerID AND IsBlocked = 1)
        RETURN -2;  -- customer is blocked

    RETURN 0;  -- success
END;

-- Capturing return value
DECLARE @ReturnCode INT;
EXEC @ReturnCode = usp_ValidateCustomer @CustomerID = 101;

SELECT CASE @ReturnCode
    WHEN  0 THEN 'Valid customer'
    WHEN -1 THEN 'Customer not found'
    WHEN -2 THEN 'Customer is blocked'
END AS ValidationResult;
```
### Parameter Best Practices
```sql
-- ✅ Always specify exact data types matching underlying columns
CREATE PROCEDURE usp_GetCustomer
    @CustomerID INT,          -- match column type exactly (avoid implicit conversion)
    @Email      VARCHAR(255)  -- match column length (avoid truncation)
AS ...

-- ✅ Use SET NOCOUNT ON to suppress unnecessary row count messages
-- This improves performance in loops and reduces network traffic

-- ✅ Use default NULL for optional parameters — check with IS NULL in body
-- ❌ Avoid VARCHAR(MAX) parameters unless necessary — they disable certain optimizations

-- ✅ Validate parameters at the start
IF @CustomerID <= 0
    THROW 50001, 'CustomerID must be positive', 1;
```
---

## 2. Error Handling (TRY/CATCH, THROW, RAISERROR)
### TRY / CATCH
SQL Server's structured error handling. Code in the `TRY` block is monitored — if any error occurs with severity > 10, execution jumps to the `CATCH` block.

```sql
BEGIN TRY
    -- Code that might fail
    BEGIN TRANSACTION;

    INSERT INTO Orders (CustomerID, Amount) VALUES (101, 500);
    INSERT INTO OrderItems (OrderID, ProductID, Quantity)
    VALUES (SCOPE_IDENTITY(), 999, 1);  -- ProductID 999 might not exist

    COMMIT TRANSACTION;
END TRY
BEGIN CATCH
    -- Handle the error
    IF @@TRANCOUNT > 0
        ROLLBACK TRANSACTION;

    -- Error information functions
    SELECT
        ERROR_NUMBER()    AS ErrorNumber,    -- e.g. 547 (FK violation)
        ERROR_SEVERITY()  AS ErrorSeverity,  -- 1-25 (>= 16 are serious)
        ERROR_STATE()     AS ErrorState,
        ERROR_PROCEDURE() AS ErrorProcedure, -- which procedure failed
        ERROR_LINE()      AS ErrorLine,
        ERROR_MESSAGE()   AS ErrorMessage;
END CATCH;
```
### Error Information Functions
```sql
-- All available inside CATCH block
ERROR_NUMBER()    -- SQL Server error number (e.g., 2627 = unique violation)
ERROR_MESSAGE()   -- Full error message text
ERROR_SEVERITY()  -- Severity level (11-16 = user errors, 17-25 = system/fatal)
ERROR_STATE()     -- State number (distinguishes similar errors)
ERROR_LINE()      -- Line number where error occurred
ERROR_PROCEDURE() -- Name of procedure/trigger/function where error occurred
```
### Common Error Numbers to Know
| Error Number | Meaning |
| ----- | ----- |
| 547 | FK constraint violation |
| 2627 | Unique constraint / PK violation |
| 2601 | Duplicate key in unique index |
| 1205 | Deadlock victim |
| 8134 | Divide by zero |
| 515 | Cannot insert NULL |
| 245 | Conversion failed |
### THROW
`THROW` is the **modern way** (SQL Server 2012+) to raise errors. It re-raises the original error with original number, severity, and state.

```sql
-- Re-raise the caught error (preserves original error details)
BEGIN TRY
    EXEC usp_RiskyOperation;
END TRY
BEGIN CATCH
    IF @@TRANCOUNT > 0 ROLLBACK;
    THROW;  -- re-raises original error to caller, no arguments needed
END CATCH;

-- Raise a custom error
THROW 50001, 'Customer not found', 1;
-- Args: error_number (>= 50000), message, state

-- THROW inside CATCH with custom message
BEGIN CATCH
    DECLARE @Msg VARCHAR(2000) =
        'Failed in ' + ISNULL(ERROR_PROCEDURE(), 'unknown') +
        ' at line ' + CAST(ERROR_LINE() AS VARCHAR) +
        ': ' + ERROR_MESSAGE();
    THROW 50099, @Msg, 1;
END CATCH;
```
### RAISERROR
`RAISERROR` is the **older method** — still commonly used and seen in legacy code. Unlike THROW, it doesn't re-raise the original error; it creates a new one.

```sql
-- Basic RAISERROR
RAISERROR('Something went wrong', 16, 1);
-- Args: message, severity, state

-- With formatting (like printf)
RAISERROR('Customer %d not found in region %s', 16, 1, @CustomerID, @Region);

-- Using a message number (pre-defined in sys.messages)
RAISERROR(50001, 16, 1);  -- uses message stored in sys.messages

-- WITH LOG — writes to SQL Server error log and Windows Event Log
RAISERROR('Critical failure occurred', 17, 1) WITH LOG;

-- WITH NOWAIT — sends message to client immediately (useful for progress)
RAISERROR('Processing batch 1 of 10...', 0, 1) WITH NOWAIT;
WAITFOR DELAY '00:00:02';
RAISERROR('Processing batch 2 of 10...', 0, 1) WITH NOWAIT;
```
### THROW vs RAISERROR
| Feature | THROW | RAISERROR |
| ----- | ----- | ----- |
| SQL Version | 2012+ | All versions |
| Re-raises original error | ✅ Yes (no args) | ❌ No — creates new error |
| Preserves error number | ✅ Yes | ❌ No (generates new) |
| String formatting | ❌ No | ✅ Yes (printf-style) |
| Severity control | ❌ Fixed at 16 | ✅ Any severity 1-25 |
| Aborts batch | ✅ Always | Only if severity >= 20 |
| Recommended | ✅ Modern code | Legacy/special cases |
| Must be in CATCH for re-raise | ✅ Yes (no-arg form) | ❌ Can use anywhere |
### Full Error Handling Pattern
```sql
CREATE PROCEDURE usp_ProcessOrder
    @OrderID    INT,
    @ProcessedBy VARCHAR(100)
AS
BEGIN
    SET NOCOUNT ON;

    -- Input validation (before transaction)
    IF @OrderID IS NULL OR @OrderID <= 0
        THROW 50001, 'Invalid OrderID provided', 1;

    IF NOT EXISTS (SELECT 1 FROM Orders WHERE OrderID = @OrderID)
        THROW 50002, 'Order not found', 1;

    BEGIN TRANSACTION;
    BEGIN TRY
        UPDATE Orders
        SET Status      = 'Processing',
            ProcessedBy = @ProcessedBy,
            ProcessedAt = GETDATE()
        WHERE OrderID = @OrderID
          AND Status  = 'Pending';

        IF @@ROWCOUNT = 0
            THROW 50003, 'Order is not in Pending status', 1;

        INSERT INTO OrderAudit (OrderID, Action, ActionBy, ActionAt)
        VALUES (@OrderID, 'PROCESS', @ProcessedBy, GETDATE());

        COMMIT TRANSACTION;
    END TRY
    BEGIN CATCH
        IF @@TRANCOUNT > 0
            ROLLBACK TRANSACTION;
        THROW;  -- propagate error to caller
    END CATCH;
END;
```
---

## 3. Dynamic SQL and sp_executesql
### What Is Dynamic SQL?
Dynamic SQL is T-SQL code that is **built as a string at runtime** and then executed. It's used when table names, column names, or conditions can't be known at compile time.

### EXEC (Simple Dynamic SQL)
```sql
-- Basic dynamic SQL with EXEC
DECLARE @TableName  NVARCHAR(128) = 'Orders';
DECLARE @SQL        NVARCHAR(500);

SET @SQL = 'SELECT TOP 10 * FROM ' + QUOTENAME(@TableName);
EXEC (@SQL);

-- QUOTENAME wraps the name in brackets: [Orders]
-- Protects against SQL injection when dealing with object names
```
### sp_executesql (Parameterized Dynamic SQL)
`sp_executesql` is the **correct way** to execute dynamic SQL with parameters. It supports parameter binding (preventing SQL injection), allows plan caching, and handles output parameters.

```sql
-- sp_executesql syntax
EXEC sp_executesql
    @statement  = N'SELECT * FROM Orders WHERE CustomerID = @CustID AND Status = @Status',
    @params     = N'@CustID INT, @Status VARCHAR(20)',  -- parameter definitions
    @CustID     = 101,                                  -- parameter values
    @Status     = 'Pending';

-- The plan for this query is cached and reused
-- CustomerID is bound as a parameter — NOT concatenated into the string
-- ✅ Safe from SQL injection
```
### Dynamic SQL with Output Parameters
```sql
DECLARE @Count      INT;
DECLARE @SQL        NVARCHAR(500);
DECLARE @Params     NVARCHAR(200);

SET @SQL    = N'SELECT @TotalCount = COUNT(*) FROM Orders WHERE Status = @Status';
SET @Params = N'@Status VARCHAR(20), @TotalCount INT OUTPUT';

EXEC sp_executesql
    @SQL,
    @Params,
    @Status     = 'Pending',
    @TotalCount = @Count OUTPUT;

SELECT @Count AS PendingOrderCount;
```
### Dynamic SQL for Variable Object Names
```sql
-- Dynamic column selection (can't parameterize column/table names)
CREATE PROCEDURE usp_GetColumnStats
    @TableName  NVARCHAR(128),
    @ColumnName NVARCHAR(128)
AS
BEGIN
    -- Validate inputs against system catalog (never trust user input directly)
    IF NOT EXISTS (
        SELECT 1 FROM sys.columns c
        JOIN sys.tables t ON c.object_id = t.object_id
        WHERE t.name = @TableName AND c.name = @ColumnName
    )
        THROW 50001, 'Table or column does not exist', 1;

    DECLARE @SQL NVARCHAR(1000);
    SET @SQL = N'SELECT
        MIN(' + QUOTENAME(@ColumnName) + N') AS MinValue,
        MAX(' + QUOTENAME(@ColumnName) + N') AS MaxValue,
        COUNT(DISTINCT ' + QUOTENAME(@ColumnName) + N') AS DistinctValues
    FROM ' + QUOTENAME(@TableName);

    EXEC sp_executesql @SQL;
END;
```
### SQL Injection — What to Avoid
```sql
-- ❌ DANGEROUS — direct string concatenation with user input
DECLARE @UserInput VARCHAR(100) = '101; DROP TABLE Orders; --';
DECLARE @SQL NVARCHAR(500) = 'SELECT * FROM Orders WHERE CustomerID = ' + @UserInput;
EXEC (@SQL);
-- Executes: SELECT * FROM Orders WHERE CustomerID = 101; DROP TABLE Orders; --
-- !! Table is dropped !!

-- ✅ SAFE — parameterized with sp_executesql
EXEC sp_executesql
    N'SELECT * FROM Orders WHERE CustomerID = @CustID',
    N'@CustID INT',
    @CustID = @UserInput;  -- value is bound, never interpreted as SQL
```
### Dynamic SQL Best Practices
```sql
-- Always use NVARCHAR (not VARCHAR) for dynamic SQL strings
-- Always use QUOTENAME() for object names (table, column, schema)
-- Always validate object names against system catalog
-- Always use sp_executesql with parameters for data values
-- Avoid EXEC() for anything that includes user-supplied values
-- Use PRINT @SQL during development to inspect the generated string

PRINT @SQL;  -- Debug: see what you're about to execute
```
### Plan Caching with sp_executesql
```sql
-- These all generate ONE cached plan (parameterized)
EXEC sp_executesql N'SELECT * FROM Orders WHERE CustomerID = @ID', N'@ID INT', 101;
EXEC sp_executesql N'SELECT * FROM Orders WHERE CustomerID = @ID', N'@ID INT', 202;
EXEC sp_executesql N'SELECT * FROM Orders WHERE CustomerID = @ID', N'@ID INT', 303;

-- vs EXEC — generates THREE separate cached plans (cache bloat)
EXEC ('SELECT * FROM Orders WHERE CustomerID = 101');
EXEC ('SELECT * FROM Orders WHERE CustomerID = 202');
EXEC ('SELECT * FROM Orders WHERE CustomerID = 303');
```
---

## 4. Cursors vs Set-Based Operations
### What Is a Cursor?
A cursor allows **row-by-row processing** of a result set — similar to a loop in a programming language. SQL Server is optimized for set-based operations; cursors bypass this and are almost always significantly slower.

```sql
-- Cursor example — update discount based on order count
DECLARE @CustomerID INT;
DECLARE @OrderCount INT;

DECLARE customer_cursor CURSOR
    LOCAL STATIC READ_ONLY FORWARD_ONLY  -- best-performing cursor options
FOR
    SELECT CustomerID FROM Customers WHERE IsActive = 1;

OPEN customer_cursor;
FETCH NEXT FROM customer_cursor INTO @CustomerID;

WHILE @@FETCH_STATUS = 0
BEGIN
    SELECT @OrderCount = COUNT(*)
    FROM Orders
    WHERE CustomerID = @CustomerID;

    UPDATE Customers
    SET Discount = CASE
        WHEN @OrderCount >= 100 THEN 0.20
        WHEN @OrderCount >= 50  THEN 0.15
        WHEN @OrderCount >= 10  THEN 0.10
        ELSE 0.00
    END
    WHERE CustomerID = @CustomerID;

    FETCH NEXT FROM customer_cursor INTO @CustomerID;
END;

CLOSE customer_cursor;
DEALLOCATE customer_cursor;
```
### Set-Based Replacement (Always Prefer This)
```sql
-- The same logic — set-based, single statement, dramatically faster
UPDATE c
SET c.Discount = CASE
    WHEN cnt.OrderCount >= 100 THEN 0.20
    WHEN cnt.OrderCount >= 50  THEN 0.15
    WHEN cnt.OrderCount >= 10  THEN 0.10
    ELSE 0.00
END
FROM Customers c
JOIN (
    SELECT CustomerID, COUNT(*) AS OrderCount
    FROM Orders
    GROUP BY CustomerID
) cnt ON c.CustomerID = cnt.CustomerID
WHERE c.IsActive = 1;
```
### Cursor Types
When you must use a cursor, choose the right type:

| Option | Description | Performance |
| ----- | ----- | ----- |
|  | Copies result set to tempdb | Snapshot of data, no reflects changes |
|  | Can only fetch forward | Fastest — no scrolling overhead |
|  | No updates through cursor | Faster — no update lock overhead |
|  | Scope limited to batch/proc | Safer — auto-deallocated |
|  | Combines FORWARD_ONLY + READ_ONLY | Best performance cursor |
```sql
-- Best performing cursor configuration
DECLARE cur CURSOR
    LOCAL         -- scoped to this procedure
    STATIC        -- snapshot copy
    FORWARD_ONLY  -- no scrolling
    READ_ONLY     -- no updates via cursor
FOR
    SELECT ...;
```
### When Cursors Are Acceptable
Despite being slow, cursors are sometimes the right tool:

- **Administrative tasks** — running a procedure for each database/table
- **Cursor output to dynamic DDL** — generating ALTER statements per table
- **Sequential dependency** — each row's processing depends on the previous row's result
- **Extremely small row counts** — processing 5–10 rows where set-based overhead isn't worth it
```sql
-- Legitimate cursor use: run DBCC on each database
DECLARE @DBName NVARCHAR(128);
DECLARE @SQL    NVARCHAR(300);

DECLARE db_cursor CURSOR LOCAL FAST_FORWARD FOR
    SELECT name FROM sys.databases WHERE state_desc = 'ONLINE';

OPEN db_cursor;
FETCH NEXT FROM db_cursor INTO @DBName;

WHILE @@FETCH_STATUS = 0
BEGIN
    SET @SQL = N'DBCC CHECKDB([' + @DBName + N']) WITH NO_INFOMSGS';
    EXEC sp_executesql @SQL;
    FETCH NEXT FROM db_cursor INTO @DBName;
END;

CLOSE db_cursor;
DEALLOCATE db_cursor;
```
### Cursor vs Set-Based Performance
```
Table: 100,000 customers
Operation: Update discount based on order count

Cursor approach:   ~45 seconds  (100,000 individual updates)
Set-based approach: ~0.3 seconds (1 set-based UPDATE with JOIN)

Ratio: 150x faster with set-based
```
---

## 5. Temp Tables (#temp) vs Table Variables (@table) vs CTEs
### Temporary Tables (#temp)
Temp tables are **real tables stored in tempdb**. They support indexes, statistics, and all table operations. Prefixed with `#` (local) or `##` (global).

```sql
-- Local temp table — visible only to current session
CREATE TABLE #OrderSummary (
    CustomerID   INT           NOT NULL,
    OrderCount   INT           NOT NULL,
    TotalAmount  DECIMAL(18,2) NOT NULL,
    LastOrderDate DATETIME     NULL,
    INDEX IX_#OrderSummary_CustomerID (CustomerID)  -- ✅ indexes supported
);

-- Populate
INSERT INTO #OrderSummary (CustomerID, OrderCount, TotalAmount, LastOrderDate)
SELECT
    CustomerID,
    COUNT(*)             AS OrderCount,
    SUM(TotalAmount)     AS TotalAmount,
    MAX(OrderDate)       AS LastOrderDate
FROM Orders
GROUP BY CustomerID;

-- Query multiple times
SELECT * FROM #OrderSummary WHERE OrderCount > 10;
SELECT * FROM #OrderSummary WHERE TotalAmount > 5000;

-- Explicitly drop (auto-dropped when session ends)
DROP TABLE IF EXISTS #OrderSummary;

-- Global temp table — visible to ALL sessions (use with caution)
CREATE TABLE ##SharedTemp (ID INT, Value VARCHAR(100));
-- Dropped when creating session ends AND no other session references it
```
### Table Variables (@table)
Table variables are **declared in memory (conceptually)** using `DECLARE`. They have limited functionality compared to temp tables — no statistics, limited indexing (SQL Server 2014+ supports non-clustered indexes inline).

```sql
-- Table variable
DECLARE @OrderSummary TABLE (
    CustomerID   INT           NOT NULL,
    OrderCount   INT           NOT NULL,
    TotalAmount  DECIMAL(18,2) NOT NULL,
    PRIMARY KEY (CustomerID)  -- primary key creates clustered index
    -- No statistics created — optimizer always estimates 1 row
);

INSERT INTO @OrderSummary
SELECT CustomerID, COUNT(*), SUM(TotalAmount)
FROM Orders
GROUP BY CustomerID;

SELECT * FROM @OrderSummary WHERE TotalAmount > 5000;
-- @OrderSummary is automatically gone after batch/procedure ends
```
### CTEs (Common Table Expressions)
CTEs are **named subqueries** defined with `WITH` — they have no physical storage and exist only for the duration of a single query. They're not stored anywhere; SQL Server inlines them into the query.

```sql
-- Single CTE
WITH OrderTotals AS (
    SELECT
        CustomerID,
        COUNT(*)         AS OrderCount,
        SUM(TotalAmount) AS TotalAmount
    FROM Orders
    GROUP BY CustomerID
)
SELECT
    c.CustomerName,
    ot.OrderCount,
    ot.TotalAmount
FROM Customers c
JOIN OrderTotals ot ON c.CustomerID = ot.CustomerID
WHERE ot.OrderCount > 10;

-- Multiple CTEs chained together
WITH
ActiveCustomers AS (
    SELECT CustomerID, CustomerName, Region
    FROM Customers
    WHERE IsActive = 1
),
CustomerOrders AS (
    SELECT CustomerID, COUNT(*) AS OrderCount, SUM(TotalAmount) AS Total
    FROM Orders
    WHERE OrderDate >= '2024-01-01'
    GROUP BY CustomerID
),
RankedCustomers AS (
    SELECT
        ac.CustomerName,
        ac.Region,
        co.OrderCount,
        co.Total,
        RANK() OVER (PARTITION BY ac.Region ORDER BY co.Total DESC) AS RegionRank
    FROM ActiveCustomers ac
    JOIN CustomerOrders co ON ac.CustomerID = co.CustomerID
)
SELECT * FROM RankedCustomers WHERE RegionRank <= 5;
```
### Comparison: Temp Tables vs Table Variables vs CTEs
| Feature | #Temp Table | @Table Variable | CTE |
| ----- | ----- | ----- | ----- |
| **Storage** | tempdb (real table) | Memory / tempdb if large | None (inline) |
| **Statistics** | ✅ Yes — good estimates | ❌ No — always estimates 1 row | Inherits from base tables |
| **Indexes** | ✅ Full index support | Limited (PK + inline NCI) | N/A |
| **Scope** | Session (until drop/disconnect) | Batch / procedure | Single query only |
| **Transaction rollback** | ✅ Rolled back | ❌ NOT rolled back | N/A |
| **Reuse in query** | ✅ Multiple queries | ✅ Multiple queries | ❌ Single statement only |
| **DDL after creation** | ✅ Yes (ALTER TABLE) | ❌ No | N/A |
| **Passes across EXEC** | ✅ Yes (visible to child procs) | ❌ No | N/A |
| **Best row count** | > 1,000 rows | < 100 rows | Variable |
| **Performance** | Better for large sets | Better for tiny sets | Depends on complexity |
### Decision Guide
```
How many rows?
├── Tiny (< 100)    → @Table Variable (low overhead, no tempdb contention)
├── Small-Medium    → CTE (if used once) or #Temp (if reused/indexed)
└── Large (> 1000)  → #Temp Table (statistics + indexes = better plan)

Used more than once in query?
├── Yes → #Temp or @Table Variable
└── No  → CTE (cleanest syntax)

Need to survive a ROLLBACK?
├── Yes → @Table Variable (not transactional)
└── No  → Either works

Passed to child stored procedure?
├── Yes → #Temp Table (shared via session)
└── No  → Any option works

Complex multi-step build-up?
└── #Temp Table (add indexes between steps for best performance)
```
### The ROLLBACK Behavior Difference
```sql
-- Critical: table variables survive ROLLBACK
BEGIN TRANSACTION;
    DECLARE @Log TABLE (Message VARCHAR(200), LoggedAt DATETIME DEFAULT GETDATE());
    INSERT INTO @Log VALUES ('Started processing');

    INSERT INTO Orders (CustomerID, Amount) VALUES (101, 500);
    INSERT INTO @Log VALUES ('Order inserted');

ROLLBACK;  -- Order INSERT is rolled back, but @Log data IS PRESERVED
           -- This is useful for logging errors even when transaction fails

SELECT * FROM @Log;  -- still returns 2 rows ✅
```
---

## 6. Recursive CTEs
A recursive CTE **references itself** to process hierarchical or tree-structured data without complex loops or cursors. It consists of an anchor member (base case) and a recursive member (repeating step).

### Structure
```sql
WITH RecursiveCTE AS (
    -- Anchor member: starting point (runs once)
    SELECT ... FROM Table WHERE <base condition>

    UNION ALL  -- must be UNION ALL, not UNION

    -- Recursive member: joins back to CTE (runs repeatedly)
    SELECT ... FROM Table
    JOIN RecursiveCTE ON <join condition>
    -- SQL Server stops when this produces no more rows
)
SELECT * FROM RecursiveCTE;
-- OPTION (MAXRECURSION n) -- default max is 100 levels
```
### Example 1: Employee Hierarchy
```sql
-- Employee table with manager relationship
-- EmployeeID | Name          | ManagerID
-- 1          | CEO           | NULL
-- 2          | VP Sales      | 1
-- 3          | VP Tech       | 1
-- 4          | Sales Manager | 2
-- 5          | Dev Lead      | 3

WITH EmployeeHierarchy AS (
    -- Anchor: start with the CEO (no manager)
    SELECT
        EmployeeID,
        Name,
        ManagerID,
        0               AS Level,
        CAST(Name AS VARCHAR(500)) AS OrgPath
    FROM Employees
    WHERE ManagerID IS NULL

    UNION ALL

    -- Recursive: find all direct reports of current level
    SELECT
        e.EmployeeID,
        e.Name,
        e.ManagerID,
        eh.Level + 1,
        CAST(eh.OrgPath + ' > ' + e.Name AS VARCHAR(500))
    FROM Employees e
    JOIN EmployeeHierarchy eh ON e.ManagerID = eh.EmployeeID
)
SELECT
    REPLICATE('  ', Level) + Name AS OrgChart,  -- indent by level
    Level,
    OrgPath
FROM EmployeeHierarchy
ORDER BY OrgPath;

-- Output:
-- CEO                         (Level 0)
--   VP Sales                  (Level 1)
--     Sales Manager           (Level 2)
--   VP Tech                   (Level 1)
--     Dev Lead                (Level 2)
```
### Example 2: Category Tree (Bill of Materials)
```sql
WITH CategoryTree AS (
    -- Anchor: top-level categories
    SELECT
        CategoryID,
        CategoryName,
        ParentCategoryID,
        CategoryName AS FullPath,
        1 AS Depth
    FROM Categories
    WHERE ParentCategoryID IS NULL

    UNION ALL

    -- Recursive: child categories
    SELECT
        c.CategoryID,
        c.CategoryName,
        c.ParentCategoryID,
        CAST(ct.FullPath + ' > ' + c.CategoryName AS VARCHAR(1000)),
        ct.Depth + 1
    FROM Categories c
    JOIN CategoryTree ct ON c.ParentCategoryID = ct.CategoryID
)
SELECT * FROM CategoryTree ORDER BY FullPath;
```
### Example 3: Date Series Generation
```sql
-- Generate a series of dates (no table needed)
DECLARE @StartDate DATE = '2024-01-01';
DECLARE @EndDate   DATE = '2024-01-31';

WITH DateSeries AS (
    SELECT @StartDate AS DateValue  -- Anchor

    UNION ALL

    SELECT DATEADD(DAY, 1, DateValue)  -- Recursive: add one day
    FROM DateSeries
    WHERE DateValue < @EndDate
)
SELECT DateValue FROM DateSeries
OPTION (MAXRECURSION 366);  -- allow up to 366 recursions (1 year of days)
```
### Example 4: Finding All Subordinates
```sql
-- Find all employees under a specific manager (any depth)
WITH Subordinates AS (
    SELECT EmployeeID, Name, 0 AS Depth
    FROM Employees
    WHERE EmployeeID = @ManagerID  -- starting manager

    UNION ALL

    SELECT e.EmployeeID, e.Name, s.Depth + 1
    FROM Employees e
    JOIN Subordinates s ON e.ManagerID = s.EmployeeID
)
SELECT * FROM Subordinates
WHERE EmployeeID <> @ManagerID  -- exclude the manager themselves
ORDER BY Depth, Name;
```
### MAXRECURSION
```sql
-- Default max recursion depth is 100
-- Increase for deep hierarchies or date ranges
SELECT * FROM DateSeries
OPTION (MAXRECURSION 0);    -- 0 = unlimited (use carefully — risk of infinite loop)

OPTION (MAXRECURSION 500);  -- explicit limit
```
>  **Preventing infinite recursion:** Always ensure the recursive member has a termination condition. For hierarchy traversal, ensure no circular references exist in the data — a row being its own ancestor will cause infinite recursion. 

```sql
-- Detect circular references before running recursive CTE
SELECT e.EmployeeID, e.Name
FROM Employees e
WHERE EXISTS (
    SELECT 1 FROM Employees m
    WHERE m.EmployeeID = e.ManagerID
      AND m.ManagerID  = e.EmployeeID
);
```
---

## 7. MERGE Statement
The `MERGE` statement performs **INSERT, UPDATE, and DELETE in a single atomic statement** based on matching rows between a source and target. It's commonly called an "upsert" operation.

### Basic Syntax
```sql
MERGE TargetTable AS target
USING SourceTable AS source
ON (target.KeyColumn = source.KeyColumn)       -- match condition

WHEN MATCHED THEN                              -- row exists in both
    UPDATE SET target.Col1 = source.Col1, ...

WHEN NOT MATCHED BY TARGET THEN               -- row in source, not in target
    INSERT (Col1, Col2) VALUES (source.Col1, source.Col2)

WHEN NOT MATCHED BY SOURCE THEN               -- row in target, not in source
    DELETE;                                    -- semicolon is required!
```
### Full Example: Sync Staging to Production
```sql
-- Sync daily sales staging table into production fact table
MERGE Sales.FactOrders AS target
USING Sales.StagingOrders AS source
    ON target.OrderID = source.OrderID

-- Update existing orders that have changed
WHEN MATCHED AND (
    target.TotalAmount <> source.TotalAmount OR
    target.Status      <> source.Status
) THEN
    UPDATE SET
        target.TotalAmount  = source.TotalAmount,
        target.Status       = source.Status,
        target.LastModified = GETDATE()

-- Insert new orders
WHEN NOT MATCHED BY TARGET THEN
    INSERT (OrderID, CustomerID, TotalAmount, Status, OrderDate, LastModified)
    VALUES (source.OrderID, source.CustomerID, source.TotalAmount,
            source.Status, source.OrderDate, GETDATE())

-- Delete orders that are no longer in staging (cancelled/removed)
WHEN NOT MATCHED BY SOURCE
     AND target.Status NOT IN ('Archived', 'Locked') THEN
    DELETE

-- Capture what happened to each row
OUTPUT
    $action               AS MergeAction,   -- 'INSERT', 'UPDATE', or 'DELETE'
    inserted.OrderID      AS NewOrderID,
    deleted.OrderID       AS OldOrderID,
    inserted.TotalAmount  AS NewAmount,
    deleted.TotalAmount   AS OldAmount;     -- semicolon required!
```
### MERGE with OUTPUT Clause
```sql
-- Capture results of all merge actions for logging
DECLARE @MergeResults TABLE (
    Action    VARCHAR(10),
    OrderID   INT,
    OldStatus VARCHAR(20),
    NewStatus VARCHAR(20)
);

MERGE Orders AS target
USING @UpdatedOrders AS source ON target.OrderID = source.OrderID

WHEN MATCHED THEN
    UPDATE SET target.Status = source.Status

WHEN NOT MATCHED BY TARGET THEN
    INSERT (OrderID, CustomerID, Status)
    VALUES (source.OrderID, source.CustomerID, source.Status)

OUTPUT
    $action,
    COALESCE(inserted.OrderID, deleted.OrderID),
    deleted.Status,
    inserted.Status
INTO @MergeResults;

-- Summarize what happened
SELECT Action, COUNT(*) AS RowCount
FROM @MergeResults
GROUP BY Action;
```
### Simple Upsert Pattern
```sql
-- Upsert: update if exists, insert if not
MERGE Customers AS target
USING (SELECT @CustomerID, @Name, @Email) AS source (CustomerID, Name, Email)
    ON target.CustomerID = source.CustomerID

WHEN MATCHED THEN
    UPDATE SET target.Name = source.Name, target.Email = source.Email

WHEN NOT MATCHED THEN
    INSERT (CustomerID, Name, Email)
    VALUES (source.CustomerID, source.Name, source.Email);
```
### MERGE Gotchas and Best Practices
**Gotcha 1: Semicolon is mandatory**

```sql
-- ❌ Missing semicolon causes syntax error
...
WHEN NOT MATCHED BY SOURCE THEN DELETE  -- error!

-- ✅ Always end MERGE with semicolon
WHEN NOT MATCHED BY SOURCE THEN DELETE;
```
**Gotcha 2: Duplicate source rows cause error**

```sql
-- If source has duplicate keys, MERGE raises an error
-- "The MERGE statement attempted to UPDATE or DELETE the same row more than once"

-- Fix: deduplicate source before MERGE
MERGE Target AS target
USING (
    SELECT CustomerID, Name, MAX(UpdatedAt) AS UpdatedAt
    FROM StagingCustomers
    GROUP BY CustomerID, Name  -- deduplicate
) AS source ON target.CustomerID = source.CustomerID
...;
```
**Gotcha 3: MERGE is not atomic by default under concurrent access**

```sql
-- Add HOLDLOCK to prevent race conditions
MERGE Target WITH (HOLDLOCK) AS target
USING Source AS source
ON target.ID = source.ID
...;
```
**Gotcha 4: All WHEN clauses are evaluated per row, in order**

```sql
-- A row can only match ONE WHEN clause
-- The first matching WHEN clause wins — order matters
WHEN MATCHED AND source.IsDeleted = 1 THEN DELETE
WHEN MATCHED THEN UPDATE SET ...      -- only reached if IsDeleted <> 1
WHEN NOT MATCHED THEN INSERT ...
```
### MERGE vs Separate INSERT/UPDATE/DELETE
| Approach | Pros | Cons |
| ----- | ----- | ----- |
| **MERGE** | Single statement, atomic, OUTPUT clause | Syntax complexity, gotchas, historical bugs |
| **Separate statements** | Simpler, easier to debug | Multiple round trips, not atomic |
| **MERGE with HOLDLOCK** | Safe for concurrent use | Lock overhead |
```sql
-- Alternative to MERGE: explicit IF EXISTS pattern (simpler, safer for simple upserts)
IF EXISTS (SELECT 1 FROM Customers WHERE CustomerID = @CustomerID)
    UPDATE Customers
    SET Name = @Name, Email = @Email
    WHERE CustomerID = @CustomerID;
ELSE
    INSERT INTO Customers (CustomerID, Name, Email)
    VALUES (@CustomerID, @Name, @Email);
```
---

## Quick Reference
| Feature | Key Point | Watch Out For |
| ----- | ----- | ----- |
| **Input Params** | Named calling recommended | Match data types to avoid implicit conversion |
| **Output Params** | Must use OUTPUT keyword in both definition and call |  |
| **TRY/CATCH** | Catches severity > 10 errors | Doesn't catch severity < 11 or compile errors |
| **THROW** | Re-raises original error, modern approach | Aborts batch; use RAISERROR for severity control |
| **RAISERROR** | Legacy, supports formatting and severity | Doesn't preserve original error number |
| **sp_executesql** | Parameterized, plan-cacheable, injection-safe | Always use over EXEC() for data values |
| **QUOTENAME()** | Safely wraps object names | Validate object names against system catalog too |
| **Cursor** | Row-by-row — slow | Almost always replaceable with set-based |
| **FAST_FORWARD cursor** | Best performing cursor type | Still slower than set-based |
| **#Temp Table** | Real table in tempdb, has statistics | tempdb contention under high concurrency |
| **@Table Variable** | No statistics — optimizer estimates 1 row | Poor performance for large row counts |
| **CTE** | Single-use named subquery, no storage | Re-evaluated each reference in the query |
| **Recursive CTE** | Hierarchies, sequences, tree traversal | MAXRECURSION limit; circular reference risk |
| **MERGE** | Upsert in one statement with OUTPUT | Semicolon required; deduplicate source; use HOLDLOCK |




<!--- Eraser file: https://app.eraser.io/workspace/kAdixbBGpFADV91AzplZ --->