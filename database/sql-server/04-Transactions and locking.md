<p><a target="_blank" href="https://app.eraser.io/workspace/AU1vnZXckyF0CvMKnySa" id="edit-in-eraser-github-link"><img alt="Edit in Eraser" src="https://firebasestorage.googleapis.com/v0/b/second-petal-295822.appspot.com/o/images%2Fgithub%2FOpen%20in%20Eraser.svg?alt=media&amp;token=968381c8-a7e7-472a-8ed6-4a6626da5501"></a></p>

# SQL Server Transactions & Locking
## Overview
Transactions and locking are the foundation of **data integrity and concurrency** in SQL Server. When multiple users read and write data simultaneously, SQL Server must ensure they don't interfere with each other in ways that corrupt data or produce wrong results. It achieves this through **transactions** (grouping operations into atomic units) and **locks** (controlling concurrent access to resources).

```
User A ──► Transaction 1 ──► Acquires Lock on Row 5 ──► Modifies Row 5
                                                              │
User B ──► Transaction 2 ──► Tries to Read Row 5 ────────── ▼
                              BLOCKED until T1 commits  Releases Lock
                              or rolls back             ──► User B reads
```
---

## 1. ACID Properties
ACID defines the four guarantees every database transaction must provide to ensure data integrity — even in the face of failures, crashes, or concurrent access.

### Atomicity
**All or nothing.** Every operation in a transaction either completes fully or is completely rolled back. There is no partial completion.

```sql
BEGIN TRANSACTION;
    UPDATE Accounts SET Balance = Balance - 500 WHERE AccountID = 1; -- debit
    UPDATE Accounts SET Balance = Balance + 500 WHERE AccountID = 2; -- credit
COMMIT TRANSACTION;
-- If the credit UPDATE fails, the debit is also rolled back
-- Money is never lost or created
```
If the server crashes after the debit but before the credit, SQL Server uses the **transaction log** to roll back the debit on recovery. The account is restored to its pre-transaction state.

### Consistency
**Data moves from one valid state to another valid state.** All constraints, rules, triggers, and cascades are enforced. A transaction cannot leave the database in a state that violates defined rules.

```sql
-- Consistency enforced by constraints
ALTER TABLE Orders ADD CONSTRAINT FK_Orders_Customers
    FOREIGN KEY (CustomerID) REFERENCES Customers(CustomerID);

-- This transaction will fail — consistency preserved
BEGIN TRANSACTION;
    INSERT INTO Orders (CustomerID, Amount)
    VALUES (99999, 500); -- CustomerID 99999 doesn't exist
    -- FK constraint fires → transaction rolls back
COMMIT TRANSACTION;
```
### Isolation
**Concurrent transactions don't interfere with each other.** Each transaction behaves as if it's running alone, regardless of other concurrent transactions. The degree of isolation is controlled by **isolation levels** (covered in the next section).

```sql
-- Transaction A sees a consistent view of data
-- even while Transaction B is modifying it simultaneously
BEGIN TRANSACTION; -- T1

    SELECT Balance FROM Accounts WHERE AccountID = 1;
    -- Returns 1000, even if T2 is currently updating this row

COMMIT TRANSACTION;
```
### Durability
**Once committed, data survives failures.** SQL Server uses **Write-Ahead Logging (WAL)** — log records are written to disk before the data pages themselves. After a COMMIT, the transaction log record is hardened to disk. Even if the server crashes immediately after, the committed data can be recovered.

```sql
COMMIT TRANSACTION;
-- After this returns, data is guaranteed durable
-- Server crash, power loss — data is safe
-- SQL Server will recover it from the transaction log on restart
```
### ACID Summary
| Property | Guarantee | Mechanism |
| ----- | ----- | ----- |
| <p>**A**</p><p>tomicity</p> | All or nothing | Transaction log + rollback |
| <p>**C**</p><p>onsistency</p> | Valid state to valid state | Constraints, triggers, rules |
| <p>**I**</p><p>solation</p> | Transactions don't interfere | Locks / Row versioning |
| <p>**D**</p><p>urability</p> | Committed data survives failures | Write-Ahead Logging (WAL) |
---

## 2. Isolation Levels
Isolation levels control **what data a transaction can see** when other transactions are simultaneously modifying that data. They define a tradeoff between consistency and concurrency.

### The Three Read Problems
| Problem | Description |
| ----- | ----- |
| **Dirty Read** | Reading uncommitted changes from another transaction |
| **Non-Repeatable Read** | Reading a row twice and getting different values |
| **Phantom Read** | Re-running a query and getting different rows in the result set |
### Read Uncommitted
The **lowest isolation level**. Transactions can read data that other transactions have modified but not yet committed. Dirty reads are allowed. No shared locks are acquired on reads.

```sql
SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;

-- Or use NOLOCK hint per table
SELECT * FROM Orders WITH (NOLOCK);
```
```
Transaction A                    Transaction B
─────────────────────────────    ─────────────────────────────
BEGIN TRAN
UPDATE Orders SET Amount = 999
WHERE OrderID = 1
-- not committed yet

                                 SELECT Amount FROM Orders
                                 WHERE OrderID = 1
                                 -- Returns 999 ← DIRTY READ!

ROLLBACK  ← amount reverted      -- B read data that never existed
```
**Use when:** Approximate reporting where slightly stale/inconsistent results are acceptable. Never use for financial, inventory, or critical business data.

**Prevents:** Nothing **Allows:** Dirty reads, non-repeatable reads, phantom reads

### Read Committed (Default)
SQL Server's **default isolation level**. Only reads committed data. Shared locks are acquired while reading a row and **released immediately** after the read — not held for the transaction duration.

```sql
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;
-- This is the default — no need to set explicitly
```
```
Transaction A                    Transaction B
─────────────────────────────    ─────────────────────────────
BEGIN TRAN
UPDATE Orders SET Amount = 999
WHERE OrderID = 1
-- Exclusive lock held

                                 SELECT Amount FROM Orders
                                 WHERE OrderID = 1
                                 -- BLOCKED by A's exclusive lock

COMMIT  ← releases exclusive lock
                                 -- Unblocked, reads committed value
```
**Prevents:** Dirty reads **Allows:** Non-repeatable reads, phantom reads

### Repeatable Read
Shared locks are held on **all rows read** for the entire transaction duration. Guarantees that if you read the same row twice, you get the same value. But new rows can still be inserted by other transactions.

```sql
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;
```
```
Transaction A (Repeatable Read)  Transaction B
─────────────────────────────    ─────────────────────────────
BEGIN TRAN
SELECT Amount FROM Orders
WHERE OrderID = 1  → 500
-- Shared lock HELD on this row

                                 UPDATE Orders SET Amount = 999
                                 WHERE OrderID = 1
                                 -- BLOCKED — A holds shared lock

SELECT Amount FROM Orders
WHERE OrderID = 1  → 500
-- Same value guaranteed ✅

COMMIT  ← releases shared lock
                                 -- B can now update
```
**Prevents:** Dirty reads, non-repeatable reads **Allows:** Phantom reads

### Serializable
The **strictest locking-based isolation level**. Uses **range locks** to prevent phantom reads. Transactions are effectively serialized — as if they ran one at a time.

```sql
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
```
```
Transaction A (Serializable)     Transaction B
─────────────────────────────    ─────────────────────────────
BEGIN TRAN
SELECT * FROM Orders
WHERE OrderDate > '2024-01-01'
-- Range lock held on this range

                                 INSERT INTO Orders
                                 (OrderDate, Amount)
                                 VALUES ('2024-06-01', 500)
                                 -- BLOCKED — range lock prevents insert

SELECT * FROM Orders
WHERE OrderDate > '2024-01-01'
-- Same rows guaranteed ✅ (no phantoms)

COMMIT
                                 -- B's insert can now proceed
```
**Prevents:** Dirty reads, non-repeatable reads, phantom reads **Allows:** Nothing — perfect consistency **Cost:** Highest blocking, most prone to deadlocks

### Snapshot Isolation
Uses **row versioning** instead of locks. Each transaction sees a consistent snapshot of the database as it existed at the **start of the transaction**. Readers never block writers; writers never block readers.

```sql
-- Enable at database level first
ALTER DATABASE MyDB SET ALLOW_SNAPSHOT_ISOLATION ON;

-- Use in transaction
SET TRANSACTION ISOLATION LEVEL SNAPSHOT;
```
```
Transaction A (Snapshot)         Transaction B
─────────────────────────────    ─────────────────────────────
BEGIN TRAN  ← snapshot taken
-- A sees DB as it was here

                                 UPDATE Orders SET Amount = 999
                                 WHERE OrderID = 1
                                 -- No blocking! Writes new version

SELECT Amount FROM Orders
WHERE OrderID = 1  → 500
-- Sees OLD version from snapshot ✅

COMMIT
```
**Update conflict:** If two snapshot transactions try to update the same row, the second one gets an error — you must handle this in application code.

```sql
-- Update conflict error in snapshot
-- Transaction 2 gets: "Snapshot isolation transaction aborted
-- due to update conflict"
```
**Prevents:** Dirty reads, non-repeatable reads, phantom reads **Cost:** tempdb storage for row versions, update conflict handling

### Isolation Level Comparison
| Isolation Level | Dirty Read | Non-Repeatable | Phantom | Blocking | Mechanism |
| ----- | ----- | ----- | ----- | ----- | ----- |
| Read Uncommitted | ✅ Possible | ✅ Possible | ✅ Possible | None | No locks |
| Read Committed | ❌ Prevented | ✅ Possible | ✅ Possible | Moderate | Short shared locks |
| Repeatable Read | ❌ Prevented | ❌ Prevented | ✅ Possible | High | Long shared locks |
| Serializable | ❌ Prevented | ❌ Prevented | ❌ Prevented | Very High | Range locks |
| Snapshot | ❌ Prevented | ❌ Prevented | ❌ Prevented | None | Row versioning |
---

## 3. RCSI (Read Committed Snapshot Isolation)
RCSI is a **database-level setting** that changes the behavior of the default Read Committed isolation level. Instead of using shared locks, reads use **row versioning** from tempdb — readers never block writers and writers never block readers.

### Enabling RCSI
```sql
-- Requires no active connections to the database
ALTER DATABASE MyDB SET READ_COMMITTED_SNAPSHOT ON;

-- Verify it's enabled
SELECT name, is_read_committed_snapshot_on
FROM sys.databases
WHERE name = 'MyDB';
```
### RCSI vs Regular Read Committed
```
Regular Read Committed:
Reader ──► Acquires shared lock ──► Waits if writer holds exclusive lock
Writer ──► Acquires exclusive lock ──► Waits if reader holds shared lock
Result: Readers and writers block each other

RCSI:
Reader ──► Reads last committed version from tempdb version store
Writer ──► Writes new version, keeps old version in tempdb
Result: Readers and writers NEVER block each other
```
```sql
-- With RCSI, this never blocks on a locked row
-- It reads the last committed version instead
SELECT * FROM Orders WHERE CustomerID = 101;
```
### RCSI vs Snapshot Isolation
| Feature | RCSI | Snapshot Isolation |
| ----- | ----- | ----- |
| Scope | Database-level setting | Transaction-level setting |
| Snapshot point | <p>Per </p><p>**statement**</p> | <p>Per </p><p>**transaction**</p> |
| Opt-in required | No (all reads use it) | Yes (SET TRANSACTION ISOLATION LEVEL SNAPSHOT) |
| Non-repeatable reads | Still possible | Prevented |
| tempdb usage | Yes | Yes |
| Update conflicts | No | Yes |
```
RCSI — snapshot at statement start:
T1: SELECT → sees committed data as of this statement's start
T1: (some time passes)
T1: SELECT again → sees committed data as of THIS new statement's start
     (may see new commits from other transactions)

Snapshot — snapshot at transaction start:
T1: BEGIN TRAN → snapshot frozen here
T1: SELECT → sees data as of BEGIN TRAN
T1: (some time passes)
T1: SELECT again → still sees same snapshot
     (ignores commits from other transactions)
```
### When to Use RCSI
RCSI is recommended as the **default setting for most OLTP databases**. It eliminates reader/writer blocking with minimal application changes — queries don't need modification. Azure SQL Database has RCSI enabled by default.

**Tradeoffs:**

- tempdb must be sized appropriately for the version store
- Long-running transactions cause version store growth
- Slightly higher CPU due to version generation
---

## 4. Lock Types: Shared, Exclusive, Update, Intent
### Shared Lock (S)
Acquired during **read operations**. Multiple transactions can hold shared locks on the same resource simultaneously — they don't block each other. A shared lock blocks exclusive locks (writes).

```sql
-- Shared lock acquired here, released after row is read (Read Committed)
SELECT * FROM Orders WHERE OrderID = 1;

-- With Repeatable Read/Serializable — shared lock held until COMMIT
```
```
Compatibility:  S + S = ✅ Compatible (both can read)
S + X = ❌ Incompatible (can't write while someone reads)
```
### Exclusive Lock (X)
Acquired during **write operations** (INSERT, UPDATE, DELETE). Only one transaction can hold an exclusive lock — no other transaction can read (with locking) or write the resource simultaneously.

```sql
-- Exclusive lock acquired, held until COMMIT or ROLLBACK
BEGIN TRANSACTION;
    UPDATE Orders SET Amount = 500 WHERE OrderID = 1;
    -- Exclusive lock held on this row
COMMIT; -- Lock released here
```
```
Compatibility:  X + S = ❌ Incompatible
X + X = ❌ Incompatible
X + U = ❌ Incompatible
```
### Update Lock (U)
A **hybrid lock** used to prevent a common deadlock pattern. When SQL Server reads a row intending to update it, it first acquires an Update lock (not exclusive). The Update lock is then promoted to Exclusive when the actual modification occurs.

```sql
-- SQL Server internally uses U lock here before promoting to X
UPDATE Orders SET Amount = 500 WHERE OrderID = 1;

-- The sequence:
-- 1. S lock → read the row (find it)
-- 2. U lock → signal intent to update (compatible with S, not with other U)
-- 3. X lock → actually modify the row
```
```
Compatibility:  U + S = ✅ Compatible (reader can still read)
U + U = ❌ Incompatible (prevents deadlock on same pattern)
U + X = ❌ Incompatible
```
**Why U locks prevent deadlocks:** Without U locks, two transactions could both acquire S locks on a row, then both try to upgrade to X locks — each waiting for the other to release their S lock → deadlock. With U locks, only one transaction can hold U at a time, so the second waits.

### Intent Locks
Intent locks are placed on **higher-level resources** (tables, pages) to signal that lower-level locks are held. They allow SQL Server to efficiently check table-level lock compatibility without scanning every row.

| Lock | Meaning |
| ----- | ----- |
| <p>**IS**</p><p> (Intent Shared)</p> | Some rows in this page/table have shared locks |
| <p>**IX**</p><p> (Intent Exclusive)</p> | Some rows in this page/table have exclusive locks |
| <p>**SIX**</p><p> (Shared with Intent Exclusive)</p> | Table is shared-locked but some rows are exclusively locked |
```sql
-- When you update one row:
-- Row level:   X lock on the specific row
-- Page level:  IX lock on the page containing the row
-- Table level: IX lock on the table

-- This lets SQL Server quickly reject a TABLOCK request
-- without checking every row for locks
```
### Lock Compatibility Matrix
|  | IS | S | U | IX | SIX | X |
| ----- | ----- | ----- | ----- | ----- | ----- | ----- |
| **IS** | ✅ | ✅ | ✅ | ✅ | ✅ | ❌ |
| **S** | ✅ | ✅ | ✅ | ❌ | ❌ | ❌ |
| **U** | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ |
| **IX** | ✅ | ❌ | ❌ | ✅ | ❌ | ❌ |
| **SIX** | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ |
| **X** | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ |
### Viewing Current Locks
```sql
-- See all current locks in the database
SELECT
    tl.resource_type,
    tl.resource_description,
    tl.request_mode,
    tl.request_status,
    tl.request_session_id,
    es.login_name,
    es.host_name,
    qt.text AS query_text
FROM sys.dm_tran_locks tl
JOIN sys.dm_exec_sessions es ON tl.request_session_id = es.session_id
CROSS APPLY sys.dm_exec_sql_text(es.most_recent_sql_handle) qt
WHERE tl.resource_database_id = DB_ID()
ORDER BY tl.request_session_id;
```
---

## 5. Deadlocks — Causes, Detection, Resolution
### What Is a Deadlock?
A deadlock occurs when **two or more transactions are each waiting for a lock held by the other** — a circular dependency with no way to proceed.

```
Transaction A                    Transaction B
─────────────────────────────    ─────────────────────────────
BEGIN TRAN                       BEGIN TRAN
Lock Row 1 ✅                    Lock Row 2 ✅

Try to lock Row 2 → WAITING      Try to lock Row 1 → WAITING
     ↑                                  │
     └──────────── Deadlock ────────────┘

Neither can proceed — SQL Server must kill one
```
SQL Server's **deadlock monitor** detects this every 5 seconds and kills the transaction with the lowest deadlock priority / least rollback cost (the "deadlock victim").

```
Error: Transaction (Process ID 55) was deadlocked on lock resources
with another process and has been chosen as the deadlock victim.
Rerun the transaction.  (Error 1205)
```
### Common Deadlock Causes
**Accessing objects in different order:**

```sql
-- Transaction A              -- Transaction B
BEGIN TRAN                    BEGIN TRAN
UPDATE Orders ...             UPDATE Customers ...
UPDATE Customers ...          UPDATE Orders ...
-- Deadlock if both run concurrently
```
**Upgrade deadlock (S lock → X lock):**

```sql
-- Both transactions read (S lock) then try to update (X lock)
-- T1: S on Row1, waiting for X on Row1 (blocked by T2's S)
-- T2: S on Row1, waiting for X on Row1 (blocked by T1's S)
```
**Missing indexes causing table scans (lock too many rows):**

```sql
-- Without index, UPDATE scans entire table
-- acquires many locks → higher chance of conflict
UPDATE Orders SET Status = 'Processed'
WHERE CustomerID = 101; -- No index on CustomerID
```
### Capturing Deadlock Information
**Method 1: System Health Extended Events (Always On)**

```sql
-- Deadlock graphs are automatically captured in system_health session
SELECT
    xdr.value('@timestamp', 'datetime2') AS deadlock_time,
    xdr.query('.') AS deadlock_graph
FROM (
    SELECT CAST(target_data AS XML) AS target_data
    FROM sys.dm_xe_session_targets t
    JOIN sys.dm_xe_sessions s ON t.event_session_address = s.address
    WHERE s.name = 'system_health'
      AND t.target_name = 'ring_buffer'
) AS data
CROSS APPLY target_data.nodes('//RingBufferTarget/event[@name="xml_deadlock_report"]') AS XEventData(xdr)
ORDER BY deadlock_time DESC;
```
**Method 2: Trace Flag 1222 (Detailed deadlock info in error log)**

```sql
DBCC TRACEON(1222, -1); -- Enable globally
-- Deadlock details written to SQL Server error log
```
**Method 3: Query Store / Extended Events**

```sql
-- Create a dedicated deadlock capture session
CREATE EVENT SESSION [Deadlock_Capture] ON SERVER
ADD EVENT sqlserver.xml_deadlock_report
ADD TARGET package0.event_file(
    SET filename = N'C:\Logs\Deadlocks.xel'
);
ALTER EVENT SESSION [Deadlock_Capture] ON SERVER STATE = START;
```
### Deadlock Resolution Strategies
**1. Consistent access order** — Always access objects in the same order across all transactions

```sql
-- Always update Orders before Customers — everywhere in your code
BEGIN TRAN
    UPDATE Orders ...
    UPDATE Customers ...
COMMIT
```
**2. Keep transactions short** — Hold locks for the minimum time necessary

```sql
-- ❌ Long transaction — holds locks while doing other work
BEGIN TRAN
    UPDATE Orders SET Status = 'Processing' WHERE OrderID = @ID;
    EXEC SendEmailNotification @ID;  -- slow! holds lock during email send
    UPDATE Orders SET Status = 'Processed' WHERE OrderID = @ID;
COMMIT

-- ✅ Short transaction — minimize lock duration
EXEC SendEmailNotification @ID;  -- do slow work outside transaction
BEGIN TRAN
    UPDATE Orders SET Status = 'Processed' WHERE OrderID = @ID;
COMMIT
```
**3. Use appropriate indexes** — Narrow the rows locked

```sql
-- Index prevents table scan, fewer rows locked, less deadlock risk
CREATE INDEX IX_Orders_CustomerID ON Orders (CustomerID);
```
**4. Use lower isolation levels or RCSI** — Readers don't block writers

```sql
ALTER DATABASE MyDB SET READ_COMMITTED_SNAPSHOT ON;
-- Readers never acquire shared locks → can't participate in deadlocks
```
**5. Set deadlock priority** — Control which transaction becomes the victim

```sql
SET DEADLOCK_PRIORITY LOW;   -- this session is preferred victim
SET DEADLOCK_PRIORITY HIGH;  -- this session survives deadlock
SET DEADLOCK_PRIORITY NORMAL; -- default
```
**6. Retry logic in application code** — Handle error 1205 gracefully

```csharp
// Always implement retry for deadlock victim error
for (int attempt = 0; attempt < 3; attempt++) {
    try {
        ExecuteTransaction();
        break;
    } catch (SqlException ex) when (ex.Number == 1205) {
        if (attempt == 2) throw;
        Thread.Sleep(100 * (attempt + 1)); // backoff
    }
}
```
---

## 6. Lock Escalation
### What Is Lock Escalation?
Lock escalation is SQL Server's mechanism to **convert many fine-grained locks (row/page) into a single coarse-grained lock (table)** to reduce memory overhead. Each lock consumes memory (~96 bytes). When too many locks accumulate, SQL Server escalates to a table lock.

### When Does It Happen?
Lock escalation triggers when:

- A transaction acquires **≥ 5,000 locks** on a single table
- Lock memory exceeds **40% of the lock memory pool**
```
Row lock 1  ┐
Row lock 2  │
Row lock 3  │  → Too many row locks →  Single TABLE lock
...         │                          (much less memory)
Row lock N  ┘
```
### The Problem with Lock Escalation
A table lock blocks **all other readers and writers** on the entire table — even those accessing completely different rows. This causes unexpected blocking.

```sql
-- DELETE 10,000 rows → may trigger table lock escalation
-- → blocks everyone on Orders table
DELETE FROM Orders
WHERE OrderDate < '2020-01-01';
```
### Viewing Lock Escalation
```sql
-- Check if lock escalation is occurring
SELECT
    OBJECT_NAME(object_id)        AS TableName,
    index_id,
    lock_escalation_desc,
    lock_escalation_attempt_count,
    lock_escalation_count
FROM sys.dm_db_index_operational_stats(DB_ID(), NULL, NULL, NULL)
WHERE lock_escalation_count > 0
ORDER BY lock_escalation_count DESC;
```
### Preventing Lock Escalation
**Option 1: Batch large operations**

```sql
-- Instead of one big DELETE (triggers escalation)
DELETE FROM Orders WHERE OrderDate < '2020-01-01';

-- Batch it — stay under the 5000 lock threshold
DECLARE @RowsDeleted INT = 1;
WHILE @RowsDeleted > 0
BEGIN
    DELETE TOP (1000) FROM Orders
    WHERE OrderDate < '2020-01-01';
    SET @RowsDeleted = @@ROWCOUNT;
    WAITFOR DELAY '00:00:00.100'; -- small pause between batches
END
```
**Option 2: Disable lock escalation on specific tables**

```sql
-- Disable escalation on this table (use carefully)
ALTER TABLE Orders SET (LOCK_ESCALATION = DISABLE);

-- For partitioned tables — escalate to partition, not table
ALTER TABLE Orders SET (LOCK_ESCALATION = AUTO);
```
**Option 3: Use RCSI / Snapshot** — Version-based reads don't acquire shared locks, so only write locks can escalate, significantly reducing the risk.

---

## 7. BEGIN TRAN, COMMIT, ROLLBACK, SAVEPOINT
### Basic Transaction Control
```sql
-- Explicit transaction
BEGIN TRANSACTION;           -- or BEGIN TRAN
    -- your SQL operations
COMMIT TRANSACTION;          -- or COMMIT TRAN / COMMIT

-- On error
BEGIN TRANSACTION;
    -- your SQL operations
ROLLBACK TRANSACTION;        -- or ROLLBACK TRAN / ROLLBACK
```
### @@TRANCOUNT
Tracks the **nesting level** of transactions. COMMIT decrements it; ROLLBACK resets to 0.

```sql
SELECT @@TRANCOUNT;  -- 0 = no active transaction

BEGIN TRAN; SELECT @@TRANCOUNT;  -- 1
BEGIN TRAN; SELECT @@TRANCOUNT;  -- 2
COMMIT;     SELECT @@TRANCOUNT;  -- 1 (not committed yet!)
COMMIT;     SELECT @@TRANCOUNT;  -- 0 (now committed)

-- ROLLBACK always resets to 0 regardless of nesting
BEGIN TRAN; BEGIN TRAN;
ROLLBACK;   SELECT @@TRANCOUNT;  -- 0 (entire transaction rolled back)
```
### Error Handling with TRY/CATCH
```sql
BEGIN TRANSACTION;
BEGIN TRY
    UPDATE Accounts SET Balance = Balance - 500 WHERE AccountID = 1;
    UPDATE Accounts SET Balance = Balance + 500 WHERE AccountID = 2;

    IF @@ERROR <> 0
        THROW 50001, 'Transfer failed', 1;

    COMMIT TRANSACTION;
END TRY
BEGIN CATCH
    IF @@TRANCOUNT > 0
        ROLLBACK TRANSACTION;

    -- Log or re-raise the error
    DECLARE @ErrorMessage NVARCHAR(4000) = ERROR_MESSAGE();
    DECLARE @ErrorSeverity INT          = ERROR_SEVERITY();
    DECLARE @ErrorState INT             = ERROR_STATE();

    RAISERROR(@ErrorMessage, @ErrorSeverity, @ErrorState);
END CATCH;
```
### Named Transactions
```sql
BEGIN TRANSACTION TransferFunds;
    UPDATE Accounts SET Balance = Balance - 500 WHERE AccountID = 1;
    UPDATE Accounts SET Balance = Balance + 500 WHERE AccountID = 2;
COMMIT TRANSACTION TransferFunds;

-- Named rollback
ROLLBACK TRANSACTION TransferFunds;
```
### SAVE TRANSACTION (Savepoints)
Savepoints create a **checkpoint within a transaction** that you can roll back to without rolling back the entire transaction.

```sql
BEGIN TRANSACTION;
    INSERT INTO Orders (CustomerID, Amount) VALUES (101, 500);
    SAVE TRANSACTION AfterFirstInsert;   -- savepoint created
    INSERT INTO Orders (CustomerID, Amount) VALUES (102, 750);
    -- Something went wrong with the second insert
    ROLLBACK TRANSACTION AfterFirstInsert;  -- rolls back to savepoint
    -- First insert is still pending (not rolled back)
    -- Continue with different operation
    INSERT INTO Orders (CustomerID, Amount) VALUES (103, 300);
COMMIT TRANSACTION;  -- commits the 1st and 3rd inserts only
```
```
Timeline:
BEGIN ──► Insert 1 ──► SAVEPOINT ──► Insert 2 ──► ROLLBACK TO SAVEPOINT
                           ▲                              │
                           └──────────────────────────────┘
                           (Insert 2 undone, Insert 1 preserved)
──► Insert 3 ──► COMMIT
```
>  **Important:** ROLLBACK to a savepoint does **not** end the transaction. You must still COMMIT or ROLLBACK the entire transaction afterward. 

### Implicit vs Explicit Transactions
```sql
-- Explicit (recommended) — you control start and end
BEGIN TRANSACTION;
    UPDATE ...;
COMMIT;

-- Implicit transactions (ANSI standard, not recommended in SQL Server)
SET IMPLICIT_TRANSACTIONS ON;
UPDATE ...;  -- automatically starts a transaction
COMMIT;      -- must manually commit each statement

-- Autocommit (SQL Server default) — each statement is its own transaction
UPDATE ...;  -- automatically committed if no BEGIN TRAN
```
---

## 8. Optimistic vs Pessimistic Concurrency
### Pessimistic Concurrency
**Assume conflicts will happen** — lock resources proactively before accessing them. Other transactions must wait. Traditional locking-based approach.

```
Philosophy: "Someone WILL try to modify this while I'm reading it,
so I'll lock it first to be safe."
```
```sql
-- Pessimistic read — hold shared lock, block writers
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;
BEGIN TRANSACTION;
    SELECT * FROM Orders WHERE OrderID = 1;
    -- Shared lock held — writer is blocked

    -- ... do some processing ...

    UPDATE Orders SET Status = 'Processed' WHERE OrderID = 1;
COMMIT;
```
**Pessimistic write with explicit lock:**

```sql
BEGIN TRANSACTION;
    -- Acquire update lock immediately — signal intent to write
    SELECT * FROM Orders WITH (UPDLOCK, ROWLOCK)
    WHERE OrderID = 1;
    -- No other writer can acquire U or X lock on this row

    UPDATE Orders SET Status = 'Processed' WHERE OrderID = 1;
COMMIT;
```
### Optimistic Concurrency
**Assume conflicts are rare** — don't lock resources. Proceed freely and **check for conflicts at commit time**. If a conflict is detected, retry the transaction.

```
Philosophy: "Conflicts are rare — I'll do my work freely and
only check if someone changed the data at the end."
```
**Implementation 1: Timestamp / rowversion column**

```sql
-- Add a rowversion column (automatically updated on every change)
ALTER TABLE Orders ADD RowVersion rowversion NOT NULL;

-- Read the row and capture its rowversion
DECLARE @rv ROWVERSION;
SELECT @rv = RowVersion, * FROM Orders WHERE OrderID = 1;

-- ... do processing ...

-- Update only if rowversion hasn't changed (no one else modified it)
UPDATE Orders
SET Status = 'Processed'
WHERE OrderID = 1
  AND RowVersion = @rv;  -- optimistic check

IF @@ROWCOUNT = 0
    -- 0 rows updated = conflict! someone else changed it
    RAISERROR('Conflict detected — please retry', 16, 1);
```
**Implementation 2: Snapshot Isolation (database-level optimistic)**

```sql
SET TRANSACTION ISOLATION LEVEL SNAPSHOT;
BEGIN TRANSACTION;
    SELECT * FROM Orders WHERE OrderID = 1;
    -- No lock acquired — reads from version store

    UPDATE Orders SET Status = 'Processed' WHERE OrderID = 1;
    -- SQL Server checks if row was modified since transaction started
    -- If yes → Update conflict error, transaction aborted
COMMIT;
```
**Implementation 3: Application-level (check before write)**

```sql
BEGIN TRANSACTION;
-- Check current state before updating
IF EXISTS (
    SELECT 1 FROM Orders
    WHERE OrderID = 1 AND Status = @OriginalStatus
)
BEGIN
    UPDATE Orders SET Status = 'Processed' WHERE OrderID = 1;
    COMMIT;
END
ELSE
BEGIN
    ROLLBACK;
    -- Handle conflict
END
```
### Pessimistic vs Optimistic Comparison
| Aspect | Pessimistic | Optimistic |
| ----- | ----- | ----- |
| **Philosophy** | Prevent conflicts with locks | Detect conflicts at commit |
| **Blocking** | High — readers/writers block | Low — minimal locking |
| **Throughput** | Lower (waiting) | Higher (no waiting) |
| **Conflict handling** | Automatic (wait in queue) | Manual (retry logic required) |
| **Best for** | High-contention, write-heavy | Low-contention, read-heavy |
| **Deadlock risk** | Higher | Lower |
| **Implementation** | Built-in (SQL Server default) | Requires rowversion or Snapshot |
| **Application complexity** | Low | Higher (retry logic needed) |
### When to Use Each
**Use Pessimistic when:**

- High contention — multiple users frequently modify the same rows
- Conflicts would be expensive to retry (long transactions)
- Financial systems where you need guaranteed serialization
**Use Optimistic when:**

- Read-heavy workloads with rare write conflicts
- Many users read but few write the same data
- Web applications where each request is a short transaction
- You want to scale read throughput without blocking
---

## Putting It All Together — Concurrency Decision Map
```
What is your workload?
│
├── Read-heavy, rare write conflicts?
│         └── Use RCSI (Read Committed Snapshot)
│             Readers never block, no code changes needed
│
├── Need consistent snapshot across long transaction?
│         └── Use SNAPSHOT ISOLATION
│             Add retry logic for update conflicts
│
├── High write contention, need to serialize?
│         └── Use SERIALIZABLE + keep transactions short
│             Accept blocking overhead
│
├── Reporting / analytics, OK with slightly stale data?
│         └── Use READ UNCOMMITTED / NOLOCK
│             Accept dirty reads
│
└── Mixed OLTP — default behavior is fine?
          └── Stick with READ COMMITTED (default)
              Enable RCSI to reduce blocking
```
---

## Quick Reference
| Concept | Key Point |
| ----- | ----- |
| **ACID** | Atomicity, Consistency, Isolation, Durability |
| **Shared Lock** | Read lock — compatible with other shared locks |
| **Exclusive Lock** | Write lock — incompatible with everything |
| **Update Lock** | Pre-write lock — prevents upgrade deadlocks |
| **Intent Lock** | Signals lower-level locks exist on page/table |
| **Read Uncommitted** | Dirty reads allowed, no shared locks |
| **Read Committed** | Default — only read committed data |
| **RCSI** | Read Committed with row versioning — no blocking |
| **Repeatable Read** | Hold shared locks — same row reads same value |
| **Serializable** | Range locks — no phantoms, maximum blocking |
| **Snapshot** | Row versioning — consistent transaction snapshot |
| **Deadlock** | Circular lock dependency — one victim is chosen |
| **Lock Escalation** | Row locks → table lock when too many accumulate |
| **Savepoint** | Partial rollback point within a transaction |
| **Pessimistic** | Lock first, prevent conflicts |
| **Optimistic** | Work freely, detect conflicts at commit |




<!--- Eraser file: https://app.eraser.io/workspace/AU1vnZXckyF0CvMKnySa --->