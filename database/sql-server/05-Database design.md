<p><a target="_blank" href="https://app.eraser.io/workspace/3m9qANkTctLP33TdHKzK" id="edit-in-eraser-github-link"><img alt="Edit in Eraser" src="https://firebasestorage.googleapis.com/v0/b/second-petal-295822.appspot.com/o/images%2Fgithub%2FOpen%20in%20Eraser.svg?alt=media&amp;token=968381c8-a7e7-472a-8ed6-4a6626da5501"></a></p>

# SQL Server Database Design & Objects
## Overview
Database design is the foundation everything else rests on. A well-designed schema makes queries fast, maintenance easy, and data integrity automatic. A poorly designed schema forces developers to work around it forever — with compensating indexes, complex queries, and application-level data validation that belongs in the database.

```
Bad Design                        Good Design
──────────────────────────────    ──────────────────────────────
OrderID | Customer | Items        Orders       Customers
1       | John,NY  | A,B,C   →   ──────────   ──────────
2       | Jane,LA  | D            OrderItems   Addresses
                                  ──────────   ──────────
Queries are painful               Queries are natural
Updates cause anomalies           Updates are clean
Reporting is guesswork            Reporting is straightforward
```
---

## 1. Normalization (1NF, 2NF, 3NF, BCNF)
Normalization is the process of structuring a relational database to **reduce data redundancy and improve data integrity** by organizing data into well-defined tables with clear relationships.

### First Normal Form (1NF)
**Rule:** Every column must contain **atomic (indivisible) values**, and each column must contain values of a **single type**. No repeating groups or arrays.

**Violation — multiple values in one column:**

```
StudentID | Name  | Courses
──────────────────────────────
1         | Alice | Math, Science, English   ← NOT 1NF (multi-valued)
2         | Bob   | Math, History
```
**Fixed — 1NF compliant:**

```
StudentID | Name  | Course
──────────────────────────────
1         | Alice | Math
1         | Alice | Science
1         | Alice | English
2         | Bob   | Math
2         | Bob   | History
```
```sql
-- 1NF violation in SQL Server
CREATE TABLE Students_Bad (
    StudentID INT,
    Name      VARCHAR(100),
    Courses   VARCHAR(500)  -- ❌ comma-separated list
);

-- 1NF compliant
CREATE TABLE Students (StudentID INT, Name VARCHAR(100));
CREATE TABLE StudentCourses (
    StudentID INT,
    Course    VARCHAR(100),
    PRIMARY KEY (StudentID, Course)
);
```
### Second Normal Form (2NF)
**Rule:** Must be in 1NF, and every **non-key column must depend on the ENTIRE primary key** — not just part of it. This only applies to tables with **composite primary keys**.

**Violation — partial dependency:**

```
OrderID | ProductID | ProductName | Quantity
─────────────────────────────────────────────
PK: (OrderID, ProductID)

ProductName depends ONLY on ProductID — not on the full (OrderID, ProductID) key
This is a partial dependency → NOT 2NF
```
**Fixed — 2NF compliant:**

```sql
-- Split into two tables
CREATE TABLE OrderItems (
    OrderID   INT,
    ProductID INT,
    Quantity  INT,
    PRIMARY KEY (OrderID, ProductID),
    FOREIGN KEY (ProductID) REFERENCES Products(ProductID)
);

CREATE TABLE Products (
    ProductID   INT PRIMARY KEY,
    ProductName VARCHAR(100)  -- depends only on ProductID ✅
);
```
### Third Normal Form (3NF)
**Rule:** Must be in 2NF, and **no non-key column should depend on another non-key column** (no transitive dependencies).

**Violation — transitive dependency:**

```
EmployeeID | DeptID | DeptName | DeptLocation
──────────────────────────────────────────────
PK: EmployeeID

DeptName and DeptLocation depend on DeptID, not on EmployeeID
DeptID → DeptName → transitive dependency → NOT 3NF
```
**Fixed — 3NF compliant:**

```sql
CREATE TABLE Employees (
    EmployeeID INT PRIMARY KEY,
    Name       VARCHAR(100),
    DeptID     INT,
    FOREIGN KEY (DeptID) REFERENCES Departments(DeptID)
);

CREATE TABLE Departments (
    DeptID       INT PRIMARY KEY,
    DeptName     VARCHAR(100),   -- depends on DeptID only ✅
    DeptLocation VARCHAR(100)    -- depends on DeptID only ✅
);
```
### Boyce-Codd Normal Form (BCNF)
**Rule:** A stricter version of 3NF. For every **functional dependency X → Y**, X must be a **superkey** (a key that uniquely identifies a row). BCNF eliminates anomalies that 3NF misses when there are multiple overlapping candidate keys.

**Violation — 3NF but not BCNF:**

```
Student | Course  | Teacher
────────────────────────────
Alice   | Math    | Mr. Smith
Alice   | Science | Ms. Jones
Bob     | Math    | Mr. Brown

Rules:
- Each student-course pair has one teacher
- Each teacher teaches only one course
- Multiple teachers can teach the same course

Candidate keys: (Student, Course) and (Student, Teacher)
Dependency: Teacher → Course (a non-key determines part of another key)
This violates BCNF
```
**Fixed — BCNF compliant:**

```sql
-- Split the dependency
CREATE TABLE TeacherCourse (
    TeacherID  INT PRIMARY KEY,
    TeacherName VARCHAR(100),
    Course      VARCHAR(100)   -- Teacher → Course (Teacher is a key now)
);

CREATE TABLE StudentTeacher (
    StudentID  INT,
    TeacherID  INT,
    PRIMARY KEY (StudentID, TeacherID)
);
```
### Normal Forms Summary
| Normal Form | Eliminates | Requires |
| ----- | ----- | ----- |
| **1NF** | Repeating groups, multi-valued columns | Atomic values, single type per column |
| **2NF** | Partial dependencies | 1NF + all non-keys depend on whole PK |
| **3NF** | Transitive dependencies | 2NF + no non-key depends on non-key |
| **BCNF** | Anomalies with multiple candidate keys | 3NF + every determinant is a superkey |
>  **Practical rule:** Design to 3NF for most OLTP systems. BCNF is important when you have multiple overlapping candidate keys — a less common but tricky scenario. 

---

## 2. Denormalization and When to Use It
### What Is Denormalization?
Denormalization is the **deliberate introduction of redundancy** into a normalized schema to improve **read performance**. You're trading storage and write complexity for faster queries.

```
Normalized (3NF):              Denormalized:
Orders + Customers             Orders (with CustomerName embedded)
────────────────               ────────────────────────────────────
JOIN needed for every query    No JOIN needed — data is already there
Clean writes                   Must update CustomerName in two places
No redundancy                  Redundant data (risk of inconsistency)
```
### Common Denormalization Techniques
**Storing computed/aggregated values:**

```sql
-- Normalized: calculate total on every query
SELECT o.OrderID, SUM(oi.Quantity * oi.UnitPrice) AS Total
FROM Orders o
JOIN OrderItems oi ON o.OrderID = oi.OrderID
GROUP BY o.OrderID;

-- Denormalized: store total in Orders table
ALTER TABLE Orders ADD TotalAmount DECIMAL(18,2);

-- Update via trigger when items change
CREATE TRIGGER trg_UpdateOrderTotal
ON OrderItems AFTER INSERT, UPDATE, DELETE
AS
    UPDATE o
    SET TotalAmount = (
        SELECT SUM(Quantity * UnitPrice)
        FROM OrderItems
        WHERE OrderID = o.OrderID
    )
    FROM Orders o
    WHERE o.OrderID IN (
        SELECT OrderID FROM inserted
        UNION
        SELECT OrderID FROM deleted
    );
```
**Copying frequently joined columns:**

```sql
-- Add CustomerName to Orders to avoid JOIN with Customers
ALTER TABLE Orders ADD CustomerName VARCHAR(100);

-- Now reporting queries are simpler and faster
SELECT OrderID, OrderDate, CustomerName, TotalAmount
FROM Orders
WHERE OrderDate > '2024-01-01';
-- No JOIN needed ✅
```
**Flattening hierarchical data:**

```sql
-- Normalized: self-referencing hierarchy (recursive CTEs needed)
CREATE TABLE Categories (
    CategoryID       INT PRIMARY KEY,
    ParentCategoryID INT NULL,
    Name             VARCHAR(100)
);

-- Denormalized: store full path
ALTER TABLE Categories ADD FullPath VARCHAR(500);
-- Electronics > Computers > Laptops
-- Faster filtering, simpler queries
```
### When to Denormalize
| Scenario | Reason |
| ----- | ----- |
| Read-heavy reporting/analytics | Avoid expensive JOINs on large tables |
| Frequently joined columns | Eliminate repeated JOIN patterns |
| Pre-computed aggregates | Avoid GROUP BY on millions of rows |
| Data warehouses (star/snowflake schema) | Optimized for reads, not writes |
| Cached/archive data | Data no longer changes — redundancy is safe |
### When NOT to Denormalize
- **OLTP systems with frequent updates** — redundant data gets out of sync
- **Early in design** — normalize first, denormalize only when profiling shows need
- **Without measuring** — always benchmark before and after denormalization
>  **Rule:** Normalize until it hurts (slow queries), then denormalize until it works. Never denormalize speculatively — measure first. 

---

## 3. Primary Key vs Unique Key vs Foreign Key
### Primary Key
A primary key **uniquely identifies each row** in a table. It enforces entity integrity. Rules: must be unique, cannot be NULL, only one per table.

```sql
CREATE TABLE Customers (
    CustomerID   INT           NOT NULL,
    Email        VARCHAR(255)  NOT NULL,
    CustomerName VARCHAR(100)  NOT NULL,
    CONSTRAINT PK_Customers PRIMARY KEY (CustomerID)
);

-- Composite primary key
CREATE TABLE OrderItems (
    OrderID   INT NOT NULL,
    ProductID INT NOT NULL,
    Quantity  INT NOT NULL,
    CONSTRAINT PK_OrderItems PRIMARY KEY (OrderID, ProductID)
);
```
**By default**, a primary key creates a **clustered index** in SQL Server. You can override this:

```sql
-- Primary key as non-clustered (when you want a different clustered index)
CREATE TABLE Orders (
    OrderID   INT          NOT NULL,
    OrderDate DATETIME     NOT NULL,
    CONSTRAINT PK_Orders PRIMARY KEY NONCLUSTERED (OrderID),
    INDEX CIX_Orders_Date CLUSTERED (OrderDate)  -- clustered on date instead
);
```
### Unique Key (Unique Constraint)
Enforces uniqueness on a column or combination of columns, but **unlike primary key**:

- Allows **one NULL** value (NULL is considered distinct from all other values)
- Multiple unique constraints per table
- Creates a **non-clustered index** by default
```sql
CREATE TABLE Customers (
    CustomerID INT          PRIMARY KEY,
    Email      VARCHAR(255) NOT NULL,
    NationalID VARCHAR(20)  NULL,
    CONSTRAINT UQ_Customers_Email     UNIQUE (Email),      -- no duplicate emails
    CONSTRAINT UQ_Customers_NationalID UNIQUE (NationalID)  -- allows one NULL
);

-- Composite unique constraint
CREATE TABLE ProductTranslations (
    ProductID   INT,
    LanguageCode CHAR(2),
    Description  NVARCHAR(500),
    CONSTRAINT UQ_ProductTranslations UNIQUE (ProductID, LanguageCode)
);
```
### Foreign Key
A foreign key **enforces referential integrity** — it ensures that a value in one table corresponds to an existing value in another table (parent table).

```sql
CREATE TABLE Orders (
    OrderID    INT PRIMARY KEY,
    CustomerID INT NOT NULL,
    CONSTRAINT FK_Orders_Customers
        FOREIGN KEY (CustomerID)
        REFERENCES Customers(CustomerID)
        ON DELETE CASCADE     -- delete orders when customer is deleted
        ON UPDATE NO ACTION   -- prevent customer ID changes if orders exist
);
```
**Referential action options:**

| Action | ON DELETE | ON UPDATE | Behavior |
| ----- | ----- | ----- | ----- |
|  | Default | Default | Error if parent row is modified/deleted |
|  | Common | Rare | Propagate change to child rows |
|  | Sometimes | Sometimes | Set FK column to NULL in child |
|  | Rare | Rare | Set FK column to its default value |
**Disabling FK checks for bulk loads:**

```sql
-- Temporarily disable FK constraint (bulk insert performance)
ALTER TABLE Orders NOCHECK CONSTRAINT FK_Orders_Customers;
-- ... bulk insert ...
ALTER TABLE Orders CHECK CONSTRAINT FK_Orders_Customers;
-- Re-validate
ALTER TABLE Orders WITH CHECK CHECK CONSTRAINT FK_Orders_Customers;
```
### Primary Key vs Unique Key vs Foreign Key
| Feature | Primary Key | Unique Key | Foreign Key |
| ----- | ----- | ----- | ----- |
| Purpose | Identify each row | Prevent duplicates | Enforce relationship |
| NULL allowed | ❌ Never | ✅ One NULL | Depends on column |
| Count per table | One | Many | Many |
| Creates index | Clustered (default) | Non-clustered | No index (create manually!) |
| Uniqueness | Always | Always | Not required |
>  **Important:** Foreign keys do NOT automatically create indexes in SQL Server. Always manually create an index on FK columns — otherwise joins and cascades will do table scans. 

```sql
-- Always add this when creating a FK
CREATE INDEX IX_Orders_CustomerID ON Orders (CustomerID);
```
---

## 4. Constraints (CHECK, DEFAULT, NOT NULL)
Constraints are **rules enforced at the database level** to maintain data quality. They fire before INSERT/UPDATE and reject violations automatically.

### CHECK Constraint
Validates that a column value **satisfies a condition**. Applied per row.

```sql
-- Simple value range
ALTER TABLE Products
ADD CONSTRAINT CHK_Products_Price
    CHECK (Price > 0);

-- Multiple conditions
ALTER TABLE Employees
ADD CONSTRAINT CHK_Employees_Salary
    CHECK (Salary >= 0 AND Salary <= 1000000);

-- Pattern validation
ALTER TABLE Customers
ADD CONSTRAINT CHK_Customers_Email
    CHECK (Email LIKE '%@%.%');

-- Cross-column check
ALTER TABLE Orders
ADD CONSTRAINT CHK_Orders_Dates
    CHECK (ShipDate IS NULL OR ShipDate >= OrderDate);

-- Using IN list
ALTER TABLE Orders
ADD CONSTRAINT CHK_Orders_Status
    CHECK (Status IN ('Pending', 'Processing', 'Shipped', 'Delivered', 'Cancelled'));
```
**Disabling and enabling CHECK constraints:**

```sql
-- Disable during data migration
ALTER TABLE Orders NOCHECK CONSTRAINT CHK_Orders_Status;
-- ... insert data ...
ALTER TABLE Orders CHECK CONSTRAINT CHK_Orders_Status;

-- Validate existing data after re-enabling
ALTER TABLE Orders WITH CHECK CHECK CONSTRAINT CHK_Orders_Status;
```
>  **Limitation:** CHECK constraints cannot reference other tables. For cross-table validation, use triggers or application logic. 

### DEFAULT Constraint
Provides an **automatic value** when no value is specified in an INSERT statement.

```sql
CREATE TABLE Orders (
    OrderID    INT           PRIMARY KEY,
    OrderDate  DATETIME      NOT NULL DEFAULT GETDATE(),
    Status     VARCHAR(20)   NOT NULL DEFAULT 'Pending',
    IsDeleted  BIT           NOT NULL DEFAULT 0,
    CreatedBy  VARCHAR(100)  NOT NULL DEFAULT SYSTEM_USER,
    RowVersion ROWVERSION    -- automatic, no default needed
);

-- Insert without specifying defaulted columns
INSERT INTO Orders (OrderID) VALUES (1);
-- OrderDate = current time, Status = 'Pending', IsDeleted = 0 ✅

-- Named default constraint (easier to manage)
ALTER TABLE Orders
ADD CONSTRAINT DF_Orders_Status DEFAULT 'Pending' FOR Status;

-- Drop a default constraint
ALTER TABLE Orders DROP CONSTRAINT DF_Orders_Status;
```
### NOT NULL Constraint
Prevents NULL values in a column — the column **must always have a value**.

```sql
CREATE TABLE Customers (
    CustomerID   INT           NOT NULL,  -- required
    Email        VARCHAR(255)  NOT NULL,  -- required
    Phone        VARCHAR(20)   NULL,      -- optional
    MiddleName   VARCHAR(50)   NULL       -- optional
);

-- Add NOT NULL to existing column (must provide default for existing NULLs)
ALTER TABLE Customers
ALTER COLUMN Email VARCHAR(255) NOT NULL;

-- Cannot make nullable if NULLs already exist
UPDATE Customers SET Email = 'unknown@example.com' WHERE Email IS NULL;
ALTER TABLE Customers ALTER COLUMN Email VARCHAR(255) NOT NULL;
```
### Constraint Management
```sql
-- View all constraints on a table
SELECT
    c.name             AS ConstraintName,
    c.type_desc        AS ConstraintType,
    col.name           AS ColumnName,
    cc.definition      AS Definition
FROM sys.objects c
LEFT JOIN sys.check_constraints cc ON c.object_id = cc.object_id
LEFT JOIN sys.columns col
    ON c.parent_object_id = col.object_id
    AND col.column_id = cc.parent_column_id
WHERE c.parent_object_id = OBJECT_ID('Orders')
  AND c.type IN ('C', 'D', 'F', 'PK', 'UQ');
```
---

## 5. Views vs Materialized Views (Indexed Views)
### Regular Views
A view is a **named, saved SELECT statement**. It has no storage of its own — every time you query a view, the underlying query executes. Think of it as a stored query, not stored data.

```sql
-- Create a view
CREATE VIEW vw_OrderSummary AS
    SELECT
        o.OrderID,
        o.OrderDate,
        c.CustomerName,
        c.Email,
        SUM(oi.Quantity * oi.UnitPrice) AS TotalAmount,
        COUNT(oi.ProductID)              AS ItemCount
    FROM Orders o
    JOIN Customers c  ON o.CustomerID  = c.CustomerID
    JOIN OrderItems oi ON o.OrderID    = oi.OrderID
    GROUP BY o.OrderID, o.OrderDate, c.CustomerName, c.Email;

-- Query the view like a table
SELECT * FROM vw_OrderSummary WHERE TotalAmount > 1000;
-- Internally: runs the full SELECT with JOINs and GROUP BY every time
```
**View options:**

```sql
-- WITH SCHEMABINDING — prevents changes to underlying tables
-- (required for indexed views)
CREATE VIEW vw_ActiveCustomers
WITH SCHEMABINDING AS
    SELECT CustomerID, CustomerName, Email
    FROM dbo.Customers        -- must use two-part name with SCHEMABINDING
    WHERE IsActive = 1;

-- WITH ENCRYPTION — hides the view definition
CREATE VIEW vw_SensitiveData
WITH ENCRYPTION AS
    SELECT * FROM SensitiveTable;

-- WITH CHECK OPTION — prevents inserts/updates through view
-- that would violate the view's WHERE clause
CREATE VIEW vw_ActiveOrders
WITH SCHEMABINDING AS
    SELECT OrderID, CustomerID, Status
    FROM dbo.Orders
    WHERE Status != 'Cancelled'
WITH CHECK OPTION;  -- can't insert/update to Status = 'Cancelled' through this view
```
### Indexed Views (Materialized Views)
An indexed view has a **unique clustered index** created on it, which forces SQL Server to **physically store and maintain the view's result set** on disk. The data is kept in sync automatically as underlying tables change.

```sql
-- Step 1: Create view WITH SCHEMABINDING (required)
CREATE VIEW vw_SalesByRegion
WITH SCHEMABINDING AS
    SELECT
        c.Region,
        YEAR(o.OrderDate)             AS SaleYear,
        COUNT_BIG(*)                  AS OrderCount,   -- COUNT_BIG required
        SUM(oi.Quantity * oi.UnitPrice) AS TotalSales
    FROM dbo.Orders o
    JOIN dbo.Customers c  ON o.CustomerID  = c.CustomerID
    JOIN dbo.OrderItems oi ON o.OrderID    = oi.OrderID
    GROUP BY c.Region, YEAR(o.OrderDate);

-- Step 2: Create unique clustered index (this materializes the view)
CREATE UNIQUE CLUSTERED INDEX CIX_vw_SalesByRegion
ON vw_SalesByRegion (Region, SaleYear);

-- Step 3: Query — SQL Server may use the indexed view automatically
SELECT Region, SaleYear, TotalSales
FROM vw_SalesByRegion
WHERE Region = 'North';
-- Reads from pre-computed result — very fast ✅
```
**Forcing indexed view usage (non-Enterprise editions):**

```sql
-- On Standard edition, must hint the view explicitly
SELECT Region, SaleYear, TotalSales
FROM vw_SalesByRegion WITH (NOEXPAND)  -- force use of materialized data
WHERE Region = 'North';
```
### Indexed View Restrictions
```sql
-- These are NOT allowed in indexed views:
-- ❌ Subqueries or outer references
-- ❌ TOP, DISTINCT, UNION, EXCEPT, INTERSECT
-- ❌ Non-deterministic functions (GETDATE(), NEWID(), etc.)
-- ❌ Float columns in GROUP BY
-- ❌ LEFT/RIGHT/FULL OUTER JOIN
-- ❌ * (SELECT *) — must name columns explicitly
-- ❌ COUNT(*) — must use COUNT_BIG(*)
```
### Views vs Indexed Views
| Feature | Regular View | Indexed View |
| ----- | ----- | ----- |
| Stores data | ❌ No — query executes every time | ✅ Yes — pre-computed on disk |
| Query performance | Same as underlying query | Dramatically faster for aggregations |
| Write overhead | None | Updates maintained automatically |
| Storage | None | Significant (stores result set) |
| Restrictions | Minimal | Many (deterministic, SCHEMABINDING, etc.) |
| Best for | Code reuse, security, simplification | Pre-aggregated reporting data |
---

## 6. Stored Procedures vs Functions vs Triggers
### Stored Procedures
A stored procedure is a **named, compiled batch of T-SQL** that can accept parameters, perform any DML/DDL, manage transactions, and return multiple result sets.

```sql
CREATE PROCEDURE usp_TransferFunds
    @FromAccountID INT,
    @ToAccountID   INT,
    @Amount        DECIMAL(18,2),
    @Success       BIT OUTPUT
AS
BEGIN
    SET NOCOUNT ON;

    BEGIN TRANSACTION;
    BEGIN TRY
        IF @Amount <= 0
            THROW 50001, 'Amount must be positive', 1;

        UPDATE Accounts
        SET Balance = Balance - @Amount
        WHERE AccountID = @FromAccountID
          AND Balance >= @Amount;

        IF @@ROWCOUNT = 0
            THROW 50002, 'Insufficient funds', 1;

        UPDATE Accounts
        SET Balance = Balance + @Amount
        WHERE AccountID = @ToAccountID;

        COMMIT TRANSACTION;
        SET @Success = 1;
    END TRY
    BEGIN CATCH
        ROLLBACK TRANSACTION;
        SET @Success = 0;
        THROW; -- re-raise error
    END CATCH;
END;

-- Execute
DECLARE @Result BIT;
EXEC usp_TransferFunds 1, 2, 500.00, @Result OUTPUT;
SELECT @Result AS Success;
```
### Functions
Functions **return a value** and can be used inside SELECT statements. They cannot perform DML (INSERT/UPDATE/DELETE) on base tables or manage transactions.

**Scalar Function — returns a single value:**

```sql
CREATE FUNCTION fn_GetFullName
(
    @FirstName VARCHAR(50),
    @LastName  VARCHAR(50)
)
RETURNS VARCHAR(101)
AS
BEGIN
    RETURN LTRIM(RTRIM(@FirstName)) + ' ' + LTRIM(RTRIM(@LastName));
END;

-- Usage
SELECT fn_GetFullName(FirstName, LastName) AS FullName
FROM Employees;
```
>  ⚠️ **Scalar functions are often a performance trap.** They execute once per row and prevent parallelism. For large queries, rewrite as inline expressions or inline TVFs (see next section). 

**Table-Valued Functions — returns a table (see next section)**

### Triggers
Triggers are **automatically fired code** that runs in response to DML events (INSERT, UPDATE, DELETE) on a table or view. They run within the same transaction as the triggering statement.

**AFTER Trigger (most common):**

```sql
-- Audit trigger — log all changes to Employees
CREATE TABLE EmployeeAudit (
    AuditID    INT IDENTITY PRIMARY KEY,
    EmployeeID INT,
    Action     VARCHAR(10),
    OldSalary  DECIMAL(18,2),
    NewSalary  DECIMAL(18,2),
    ChangedBy  VARCHAR(100),
    ChangedAt  DATETIME DEFAULT GETDATE()
);

CREATE TRIGGER trg_Employees_Audit
ON Employees
AFTER UPDATE
AS
BEGIN
    SET NOCOUNT ON;
    INSERT INTO EmployeeAudit (EmployeeID, Action, OldSalary, NewSalary, ChangedBy)
    SELECT
        i.EmployeeID,
        'UPDATE',
        d.Salary,
        i.Salary,
        SYSTEM_USER
    FROM inserted i
    JOIN deleted d ON i.EmployeeID = d.EmployeeID
    WHERE i.Salary <> d.Salary;  -- only log salary changes
END;
```
**INSTEAD OF Trigger — replaces the operation:**

```sql
-- Soft delete via INSTEAD OF DELETE
CREATE TRIGGER trg_Customers_SoftDelete
ON Customers
INSTEAD OF DELETE
AS
BEGIN
    SET NOCOUNT ON;
    UPDATE c
    SET IsDeleted = 1, DeletedAt = GETDATE()
    FROM Customers c
    JOIN deleted d ON c.CustomerID = d.CustomerID;
    -- Actual DELETE never happens — replaced by UPDATE
END;
```
### Stored Procedures vs Functions vs Triggers
| Feature | Stored Procedure | Scalar Function | Trigger |
| ----- | ----- | ----- | ----- |
| Returns | Result sets, output params | Single value | Nothing |
| Used in SELECT | ❌ No | ✅ Yes | N/A (auto-fired) |
| DML on base tables | ✅ Yes | ❌ No | ✅ Yes |
| Transaction control | ✅ Yes | ❌ No | Inherited |
| Called explicitly | ✅ Yes (EXEC) | ✅ Yes (inline) | ❌ No (automatic) |
| Error handling | TRY/CATCH | Limited | TRY/CATCH |
| Performance impact | Low overhead | Per-row overhead | Per-DML overhead |
| Best for | Business logic, transactions | Reusable calculations | Auditing, enforcement |
---

## 7. Inline TVF vs Multi-Statement TVF
Table-Valued Functions (TVFs) return a **table result** instead of a scalar value. There are two fundamentally different types with very different performance characteristics.

### Inline Table-Valued Function (iTVF)
An inline TVF contains a **single SELECT statement**. The optimizer treats it like a **parameterized view** — it can see inside the function, push predicates into it, use indexes, and build an optimal plan.

```sql
-- Inline TVF — single SELECT, no BEGIN/END, no RETURN variable
CREATE FUNCTION fn_GetOrdersByCustomer
(
    @CustomerID INT,
    @FromDate   DATETIME
)
RETURNS TABLE
AS
RETURN
(
    SELECT
        o.OrderID,
        o.OrderDate,
        SUM(oi.Quantity * oi.UnitPrice) AS Total
    FROM Orders o
    JOIN OrderItems oi ON o.OrderID = oi.OrderID
    WHERE o.CustomerID = @CustomerID
      AND o.OrderDate >= @FromDate
    GROUP BY o.OrderID, o.OrderDate
);

-- Usage — treated like a table, optimizer can push predicates
SELECT * FROM fn_GetOrdersByCustomer(101, '2024-01-01')
WHERE Total > 500;
-- Optimizer can push WHERE Total > 500 into the function ✅
```
### Multi-Statement Table-Valued Function (mTVF)
An mTVF uses **multiple T-SQL statements**, declares a return table variable, populates it, and returns it. The optimizer treats it as a **black box** — it cannot see inside, always estimates 1 or 100 rows, and cannot use indexes within it.

```sql
-- Multi-statement TVF — BEGIN/END, declares return table
CREATE FUNCTION fn_GetCustomerStats
(
    @Region VARCHAR(50)
)
RETURNS @Results TABLE (
    CustomerID   INT,
    CustomerName VARCHAR(100),
    TotalOrders  INT,
    TotalSpend   DECIMAL(18,2),
    Tier         VARCHAR(20)
)
AS
BEGIN
    -- Statement 1: Insert base data
    INSERT INTO @Results (CustomerID, CustomerName, TotalOrders, TotalSpend)
    SELECT
        c.CustomerID,
        c.CustomerName,
        COUNT(o.OrderID),
        SUM(oi.Quantity * oi.UnitPrice)
    FROM Customers c
    LEFT JOIN Orders o    ON c.CustomerID = o.CustomerID
    LEFT JOIN OrderItems oi ON o.OrderID  = oi.OrderID
    WHERE c.Region = @Region
    GROUP BY c.CustomerID, c.CustomerName;

    -- Statement 2: Update tier based on spend
    UPDATE @Results
    SET Tier = CASE
        WHEN TotalSpend > 10000 THEN 'Platinum'
        WHEN TotalSpend > 5000  THEN 'Gold'
        WHEN TotalSpend > 1000  THEN 'Silver'
        ELSE 'Bronze'
    END;

    RETURN;  -- returns @Results table
END;

-- Usage
SELECT * FROM fn_GetCustomerStats('North');
```
### Inline TVF vs Multi-Statement TVF
| Feature | Inline TVF | Multi-Statement TVF |
| ----- | ----- | ----- |
| Structure | Single SELECT | Multiple statements + table variable |
| Optimizer visibility | ✅ Full — treated as view | ❌ Black box |
| Cardinality estimation | Accurate | Always 1 or 100 rows (bad!) |
| Predicate pushdown | ✅ Yes | ❌ No |
| Parallelism | ✅ Yes | ❌ No (table variable) |
| Index use inside function | ✅ Yes | Limited (table variable) |
| Complex logic | ❌ Single statement only | ✅ Multiple operations |
| Performance | ⭐⭐⭐⭐⭐ | ⭐⭐ |
```
Rule: Prefer inline TVF whenever possible.
Use multi-statement TVF only when logic genuinely requires
multiple steps that can't be expressed in one SELECT.
```
>  **SQL Server 2019+ improvement:** Interleaved Execution for mTVFs — SQL Server executes the mTVF first to get actual row counts, then re-optimizes the outer query. This dramatically improves mTVF performance in modern versions. 

---

## 8. Schemas and Their Purpose
### What Is a Schema?
A schema is a **namespace/container** that groups database objects (tables, views, procedures, functions) into logical collections. Think of it as a folder inside a database.

```
Database: CompanyDB
├── Schema: dbo          (default — general objects)
├── Schema: Sales        (sales-related objects)
├── Schema: HR           (HR-related objects)
├── Schema: Reporting    (views and reporting objects)
└── Schema: Archive      (historical/archived tables)
```
### Creating and Using Schemas
```sql
-- Create schemas
CREATE SCHEMA Sales AUTHORIZATION dbo;
CREATE SCHEMA HR    AUTHORIZATION dbo;
CREATE SCHEMA Reporting;

-- Create objects in a schema
CREATE TABLE Sales.Orders (
    OrderID    INT PRIMARY KEY,
    CustomerID INT,
    OrderDate  DATETIME
);

CREATE TABLE HR.Employees (
    EmployeeID INT PRIMARY KEY,
    Name       VARCHAR(100),
    Salary     DECIMAL(18,2)
);

CREATE VIEW Reporting.vw_MonthlySales AS
    SELECT YEAR(OrderDate) AS Year, MONTH(OrderDate) AS Month,
           COUNT(*) AS Orders
    FROM Sales.Orders
    GROUP BY YEAR(OrderDate), MONTH(OrderDate);
```
### Schema-Based Security
Schemas are the primary mechanism for **permission management**. Grant permissions on a schema instead of individual objects.

```sql
-- Create users
CREATE USER SalesUser WITHOUT LOGIN;
CREATE USER HRManager WITHOUT LOGIN;
CREATE USER ReportViewer WITHOUT LOGIN;

-- Grant permissions at the schema level
GRANT SELECT, INSERT, UPDATE ON SCHEMA::Sales TO SalesUser;
GRANT SELECT, INSERT, UPDATE, DELETE ON SCHEMA::HR TO HRManager;
GRANT SELECT ON SCHEMA::Reporting TO ReportViewer;

-- SalesUser can access ALL objects in Sales schema
-- but has NO access to HR or Reporting schemas
```
### Moving Objects Between Schemas
```sql
-- Move a table to a different schema
ALTER SCHEMA Archive TRANSFER Sales.OldOrders;
-- Sales.OldOrders is now Archive.OldOrders
```
### Schema Best Practices
```sql
-- Always use two-part naming (schema.object)
SELECT * FROM Sales.Orders;         -- ✅ explicit schema
SELECT * FROM Orders;               -- ❌ implicit dbo — bad practice

-- Default schema for users
ALTER USER SalesUser WITH DEFAULT_SCHEMA = Sales;
-- Now SalesUser can write: SELECT * FROM Orders (resolves to Sales.Orders)
```
### Benefits of Schemas
| Benefit | Description |
| ----- | ----- |
| **Organization** | Group related objects logically |
| **Security** | Grant/deny access at schema level |
| **Name avoidance** | <p></p><p></p> |
| **Ownership** | Separate ownership of object groups |
| **Deployment** | Deploy/migrate schema by schema |
---

## 9. Sequences vs Identity Columns
Both generate **auto-incrementing numeric values** for primary keys, but they work very differently.

### Identity Columns
An `IDENTITY` property on a column automatically generates a new value on every INSERT. It is **tied to the table** — you can't use it independently.

```sql
-- Basic identity column
CREATE TABLE Orders (
    OrderID   INT IDENTITY(1,1) PRIMARY KEY,  -- starts at 1, increments by 1
    OrderDate DATETIME NOT NULL DEFAULT GETDATE()
);

-- Custom seed and increment
CREATE TABLE Invoices (
    InvoiceID INT IDENTITY(1000, 5) PRIMARY KEY  -- starts at 1000, increments by 5
);

-- Insert (don't specify the identity column)
INSERT INTO Orders (OrderDate) VALUES (GETDATE());
SELECT SCOPE_IDENTITY();  -- returns the generated ID for THIS scope

-- Other identity functions
SELECT @@IDENTITY;          -- last identity in current session (any table)
SELECT SCOPE_IDENTITY();    -- last identity in current scope (recommended)
SELECT IDENT_CURRENT('Orders'); -- last identity for a specific table
```
**Forcing a specific value:**

```sql
-- Override identity (for data migration)
SET IDENTITY_INSERT Orders ON;
INSERT INTO Orders (OrderID, OrderDate) VALUES (9999, GETDATE());
SET IDENTITY_INSERT Orders OFF;

-- Reset identity seed
DBCC CHECKIDENT('Orders', RESEED, 100);  -- next value will be 101
```
**Identity gaps:** Gaps occur after rollbacks — the identity value was consumed even if the row wasn't committed. This is normal and expected.

### Sequences
A `SEQUENCE` is an **independent database object** that generates numbers. It's not tied to any table — you can use the same sequence across multiple tables, or use values in application logic before inserting.

```sql
-- Create a sequence
CREATE SEQUENCE seq_OrderID
    AS INT
    START WITH 1
    INCREMENT BY 1
    MINVALUE 1
    MAXVALUE 2147483647
    CYCLE          -- restart after reaching max (remove if gaps are a problem)
    CACHE 50;      -- cache 50 values in memory for performance

-- Get next value
SELECT NEXT VALUE FOR seq_OrderID;  -- returns next number

-- Use in INSERT
INSERT INTO Orders (OrderID, OrderDate)
VALUES (NEXT VALUE FOR seq_OrderID, GETDATE());

-- Use as default constraint (most common pattern)
CREATE TABLE Orders (
    OrderID   INT DEFAULT (NEXT VALUE FOR seq_OrderID) PRIMARY KEY,
    OrderDate DATETIME
);

-- Get current/next value
SELECT current_value FROM sys.sequences WHERE name = 'seq_OrderID';
```
**Sharing sequence across tables:**

```sql
-- One sequence for multiple related tables (consistent numbering)
CREATE SEQUENCE seq_DocumentID START WITH 1 INCREMENT BY 1;

CREATE TABLE Invoices   (InvoiceID   INT DEFAULT (NEXT VALUE FOR seq_DocumentID));
CREATE TABLE CreditNotes(CreditNoteID INT DEFAULT (NEXT VALUE FOR seq_DocumentID));
-- Invoice 1, CreditNote 2, Invoice 3, CreditNote 4 — globally unique
```
### Identity vs Sequence Comparison
| Feature | IDENTITY | SEQUENCE |
| ----- | ----- | ----- |
| Scope | Single table | Independent — any use |
| Shared across tables | ❌ No | ✅ Yes |
| Use before INSERT | ❌ No | ✅ Yes (NEXT VALUE FOR) |
| Can be used in UPDATE | ❌ No | ✅ Yes |
| Transaction rollback behavior | Gaps occur (value lost) | Gaps occur (value lost) |
| Reset easily | DBCC CHECKIDENT | ALTER SEQUENCE RESTART |
| Cycling | ❌ No | ✅ Optional |
| Caching for performance | Built-in | Configurable CACHE |
| ANSI Standard | ❌ SQL Server proprietary | ✅ ANSI SQL standard |
| Use in distributed systems | ❌ Limited | ✅ Better suited |
### Sequence Cycling and Caching
```sql
-- Modify existing sequence
ALTER SEQUENCE seq_OrderID
    RESTART WITH 1        -- reset to start
    INCREMENT BY 1
    CACHE 100;            -- increase cache for performance

-- ⚠️ CACHE caveat: if SQL Server restarts, up to CACHE values are lost
-- (gaps appear). Acceptable for most use cases.
-- Use NO CACHE if no gaps is a strict requirement (slower).
ALTER SEQUENCE seq_OrderID NO CACHE;
```
---

## Design Decisions Quick Reference
```
Scenario                              Recommendation
──────────────────────────────────    ─────────────────────────────────────
OLTP system, complex business logic   Normalize to 3NF
Reporting/analytics heavy system      Denormalize or use indexed views
Unique identifier for each row        Primary Key (clustered by default)
Prevent duplicate emails/names        Unique Constraint
Link related data across tables       Foreign Key + manual index on FK col
Validate column values                CHECK Constraint
Auto-populate columns                 DEFAULT Constraint
Reusable query / simplify access      View
Pre-computed aggregation              Indexed View (WITH SCHEMABINDING)
Business logic with transactions      Stored Procedure
Reusable calculation in SELECT        Inline TVF (prefer over scalar fn)
Multi-step table transformation       Multi-Statement TVF (last resort)
Auto-logging changes                  AFTER Trigger
Override default DML behavior         INSTEAD OF Trigger
Organize and secure objects           Schemas
Simple auto-increment PK              IDENTITY column
Shared numbering / pre-fetch values   SEQUENCE object
```
---

## Summary Table
| Object | Purpose | Key Characteristic |
| ----- | ----- | ----- |
| **Primary Key** | Uniquely identify rows | NOT NULL, one per table, clustered index |
| **Unique Key** | Prevent duplicates | Allows one NULL, many per table |
| **Foreign Key** | Enforce relationships | Reference parent table, add index manually |
| **CHECK** | Validate values | Expression per row, no cross-table refs |
| **DEFAULT** | Auto-populate columns | Used when INSERT omits column |
| **View** | Saved query | No storage, executes on query |
| **Indexed View** | Materialized query | Stored on disk, auto-maintained |
| **Stored Procedure** | Business logic unit | DML, transactions, multiple result sets |
| **Scalar Function** | Single value | Used in SELECT, per-row overhead |
| **Inline TVF** | Parameterized view | Optimizer-friendly, fastest |
| **Multi-stmt TVF** | Complex table logic | Black box, poor estimation |
| **AFTER Trigger** | React to DML | Audit, cascades, denorm maintenance |
| **INSTEAD OF Trigger** | Override DML | Soft deletes, view writes |
| **Schema** | Namespace + security | Groups objects, permission boundary |
| **IDENTITY** | Auto-increment PK | Table-scoped, simple |
| **SEQUENCE** | Auto-increment number | Independent, shareable, ANSI standard |




<!--- Eraser file: https://app.eraser.io/workspace/3m9qANkTctLP33TdHKzK --->