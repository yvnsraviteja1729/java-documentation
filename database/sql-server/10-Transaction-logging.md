<p><a target="_blank" href="https://app.eraser.io/workspace/cYOIU2a2HNYjQfAqjRRx" id="edit-in-eraser-github-link"><img alt="Edit in Eraser" src="https://firebasestorage.googleapis.com/v0/b/second-petal-295822.appspot.com/o/images%2Fgithub%2FOpen%20in%20Eraser.svg?alt=media&amp;token=968381c8-a7e7-472a-8ed6-4a6626da5501"></a></p>

# SQL Server Transaction Log
## Overview
The transaction log is the **backbone of SQL Server's data integrity guarantees**. Every change to data — every INSERT, UPDATE, DELETE, DDL statement — is recorded in the transaction log before it affects data pages. It is the mechanism behind ACID properties, crash recovery, point-in-time restore, log shipping, and Always On replication.

```
Transaction Log's Roles
─────────────────────────────────────────────────────────────────
Role                    How the log is used
─────────────────────   ─────────────────────────────────────────
Crash Recovery          Redo committed changes, undo uncommitted
Point-in-Time Restore   Replay log records to any moment in time
ACID (Durability)       Committed data survives crashes (WAL)
ACID (Atomicity)        Rollback uses log to undo partial work
Always On AG            Stream log to secondary replicas
Log Shipping            Ship log backups to warm standby
Change Tracking         CDC reads changes from log
Replication             Log Reader Agent reads committed changes
─────────────────────────────────────────────────────────────────
```
---

## 1. How the Transaction Log Works
### Physical Structure
The transaction log (.ldf file) is a **sequential file** divided into **Virtual Log Files (VLFs)**. Unlike data pages which are random-access, the log is written sequentially — always appending to the current active VLF. This makes log writes extremely fast (sequential I/O).

```
Transaction Log File (.ldf)
────────────────────────────────────────────────────────────────
┌──────────┬──────────┬──────────┬──────────┬──────────┐
│  VLF 1   │  VLF 2   │  VLF 3   │  VLF 4   │  VLF 5   │
│(inactive)│(inactive)│ (active) │ (active) │  (free)  │
└──────────┴──────────┴──────────┴──────────┴──────────┘
                       ▲                    ▲
                  Oldest active         Current write
                  log record            position
                  (MinLSN)              (head of log)

Log writes happen here ──────────────────────────────────►
                                                    (circular)
```
The log is **circular** — once the inactive VLFs at the beginning are truncated (freed), the log wraps around and reuses that space.

### What Gets Written to the Log
Every modification generates one or more **log records**. Each record describes the change in enough detail to redo or undo it.

```
INSERT INTO Orders (CustomerID, Amount) VALUES (101, 500);
Generates log records:
  1. BEGIN TRANSACTION  → records start of transaction
  2. LOCK ACQUIRED      → lock information
  3. PAGE ALLOCATION    → if new page needed
  4. INSERT RECORD      → before image (nothing) + after image (new row)
  5. COMMIT             → transaction committed

UPDATE Orders SET Amount = 750 WHERE OrderID = 1;
Generates log records:
  1. BEGIN TRANSACTION
  2. UPDATE RECORD      → before image (500) + after image (750)
  3. COMMIT
```
### Log Record Contents
Each log record contains:

| Field | Description |
| ----- | ----- |
| LSN | Log Sequence Number — unique identifier |
| Transaction ID | Which transaction this belongs to |
| Operation type | LOP_INSERT_ROWS, LOP_DELETE_ROWS, LOP_MODIFY_ROW, etc. |
| Before image | Original value (used for ROLLBACK and undo) |
| After image | New value (used for REDO during recovery) |
| Object info | Page ID, file ID, slot number |
| Timestamp | When the record was written |
### Transaction Lifecycle in the Log
```
BEGIN TRANSACTION
      │
      ▼
Operations execute:
  Each change written to log buffer (memory) first
  Data pages modified in buffer pool (memory)
      │
      ▼
COMMIT TRANSACTION
      │
      ▼
Log buffer flushed to disk (.ldf) ── WAL guarantees this happens
before COMMIT returns to client    ── BEFORE data pages written to disk
      │
      ▼
COMMIT acknowledged to client ✅
      │
      ▼ (asynchronous — later)
Dirty data pages written to disk by Checkpoint / Lazy Writer

─────────────────────────────────────────────────────────
ROLLBACK path:
      │
      ▼
SQL Server reads log records BACKWARDS
Applies before images (undo each change in reverse order)
Data restored to pre-transaction state
Transaction marked as aborted in log
```
### Crash Recovery Using the Log
When SQL Server restarts after a crash, it uses the log to restore consistency — regardless of whether data pages were flushed to disk:

```
Crash Recovery Phases (ARIES Algorithm):
─────────────────────────────────────────────────────────────────
Phase 1: ANALYSIS
  - Read log from last checkpoint forward
  - Identify which transactions were active at crash time
  - Determine which pages were dirty (modified but not flushed)

Phase 2: REDO (Roll Forward)
  - Replay ALL log records from last checkpoint
  - Re-apply committed AND uncommitted changes
  - Brings database to exact state at moment of crash
  - (Even uncommitted changes are re-applied — then undone in Phase 3)

Phase 3: UNDO (Roll Back)
  - Walk backwards through log
  - Undo all uncommitted transactions (atomicity guarantee)
  - Any transaction without COMMIT record gets rolled back
─────────────────────────────────────────────────────────────────
```
```sql
-- View the transaction log contents (diagnostic)
SELECT TOP 100
    [Current LSN],
    Operation,
    Context,
    [Transaction ID],
    [Begin Time],
    [End Time],
    [Transaction Name],
    Description
FROM fn_dblog(NULL, NULL)  -- NULL, NULL = all available log records
ORDER BY [Current LSN];

-- More targeted: show only recent committed transactions
SELECT
    [Current LSN],
    Operation,
    [Transaction ID],
    [Transaction Name],
    [Begin Time]
FROM fn_dblog(NULL, NULL)
WHERE Operation IN ('LOP_BEGIN_XACT', 'LOP_COMMIT_XACT', 'LOP_ABORT_XACT')
ORDER BY [Current LSN] DESC;
```
---

## 2. Log Sequence Numbers (LSN)
### What Is an LSN?
A Log Sequence Number is a **unique, monotonically increasing identifier** assigned to every log record written to the transaction log. LSNs are the fundamental ordering mechanism — every log record has an LSN, and higher LSNs always represent later events.

```
LSN Format: file_sequence:block_offset:slot
Example:    0000002b:000001f8:0001

Broken down:
  0000002b  = VLF sequence number (which VLF)
  000001f8  = Log block (position within the VLF)
  0001      = Slot (record within the log block)
```
### How LSNs Are Used
```
Database State at any point = "Apply all log records up to LSN X"

Backup chain uses LSNs:
┌─────────────────────────────────────────────────────┐
│ Full Backup          │ Diff Backup  │ Log Backup     │
│ LSN 1 to 5000        │ LSN 5001-7000│ LSN 7001-8500  │
└─────────────────────────────────────────────────────┘

To restore to LSN 8200:
  1. Restore Full  (LSN 1-5000)
  2. Restore Diff  (LSN 5001-7000)
  3. Restore Log with STOPATLSN = 8200
```
### Key LSN Types
| LSN Type | Meaning | Where Used |
| ----- | ----- | ----- |
| **Current LSN** | Next LSN to be assigned | Log head position |
| **Min LSN (MinLSN)** | Oldest active transaction's first log record | Log truncation boundary |
| **Checkpoint LSN** | LSN of last completed checkpoint | Recovery starting point |
| **First LSN** | First LSN in a backup | Backup/restore chain |
| **Last LSN** | Last LSN in a backup | Backup/restore chain |
| **Database Fork LSN** | Point where database was forked (snapshot) | Snapshot restore |
```sql
-- View LSN information for the current database
DBCC LOGINFO;  -- VLF info including LSNs

-- Get backup LSN chain information
SELECT
    backup_start_date,
    type                              AS backup_type,
    first_lsn,
    last_lsn,
    checkpoint_lsn,
    database_backup_lsn,
    backup_size / 1024 / 1024         AS backup_mb
FROM msdb.dbo.backupset
WHERE database_name = 'OrdersDB'
ORDER BY backup_start_date DESC;

-- Check current log usage and LSN positions
SELECT
    DB_NAME()                    AS database_name,
    log_size_mb = size * 8.0 / 1024,
    log_used_mb = FILEPROPERTY(name, 'SpaceUsed') * 8.0 / 1024,
    log_used_pct = CAST(100 * (1 - ((size - FILEPROPERTY(name,'SpaceUsed'))
                   / CAST(size AS FLOAT))) AS DECIMAL(5,2))
FROM sys.database_files
WHERE type_desc = 'LOG';

-- Restore to specific LSN (point-in-time within a log backup)
RESTORE LOG OrdersDB
FROM DISK = 'D:\Backups\OrdersDB_Log.trn'
WITH NORECOVERY,
     STOPATMARK = 'lsn:0000002b:000001f8:0001';
     -- or STOPBEFOREMARK for the record before that LSN
```
### LSN in Replication and AG
```sql
-- Always On: secondary lag measured in LSNs
SELECT
    ar.replica_server_name,
    drs.log_send_queue_size,    -- KB of log not yet sent (in LSN terms)
    drs.redo_queue_size,        -- KB received but not yet applied
    drs.secondary_lag_seconds
FROM sys.dm_hadr_database_replica_states drs
JOIN sys.availability_replicas ar ON drs.replica_id = ar.replica_id;

-- Replication uses LSNs to track what's been delivered
-- Log Reader Agent reads from publisher's log starting at a specific LSN
SELECT * FROM distribution.dbo.MSrepl_transactions
ORDER BY xact_seqno DESC;  -- xact_seqno is the LSN
```
---

## 3. Log Growth and VLFs (Virtual Log Files)
### What Are VLFs?
The physical log file is internally divided into **Virtual Log Files (VLFs)**. VLFs are the unit of log management — they are activated, filled, and deactivated as the log is used. You cannot directly control VLF sizes; they are determined by the log file growth increment.

```
Log File Physical Layout:
┌────────────────────────────────────────────────────────────┐
│  Physical .ldf File                                        │
│                                                            │
│  ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐        │
│  │VLF 1│ │VLF 2│ │VLF 3│ │VLF 4│ │VLF 5│ │VLF 6│  ...   │
│  │  64 │ │  64 │ │  64 │ │  64 │ │  64 │ │  64 │        │
│  │  MB │ │  MB │ │  MB │ │  MB │ │  MB │ │  MB │        │
│  └─────┘ └─────┘ └─────┘ └─────┘ └─────┘ └─────┘        │
└────────────────────────────────────────────────────────────┘
```
### VLF Size Determination
SQL Server creates VLFs automatically based on the **growth increment size**:

| Growth Increment | VLFs Created | VLF Size |
| ----- | ----- | ----- |
| < 64 MB | 4 VLFs | Growth ÷ 4 |
| 64 MB – 1 GB | 8 VLFs | Growth ÷ 8 |
| > 1 GB | 16 VLFs | Growth ÷ 16 |
```
Example: Log file starts at 256MB, grows by 64MB each time
Initial creation (256MB):
  → 8 VLFs × 32MB each
Each 64MB auto-growth:
  → 8 VLFs × 8MB each (very small VLFs!)
After 50 auto-growths:
  256MB initial  = 8 VLFs
  50 × 64MB auto = 50 × 8 = 400 VLFs
  Total          = 408 VLFs (fragmented!)
```
### VLF Fragmentation Problem
**Too many small VLFs** (hundreds or thousands) causes:

- Slow crash recovery (SQL Server scans all VLFs during analysis phase)
- Slow database startup
- Slow log backup and restore
- Higher memory overhead
```sql
-- Check VLF count and health
DBCC LOGINFO;
-- Returns one row per VLF
-- FileId, FileSize, StartOffset, FSeqNo, Status, Parity, CreateLSN
-- Status: 0 = inactive, 2 = active

-- Count VLFs per database (quick check)
SELECT
    DB_NAME(database_id)        AS database_name,
    COUNT(*)                    AS vlf_count,
    CASE
        WHEN COUNT(*) < 50   THEN 'Healthy'
        WHEN COUNT(*) < 200  THEN 'Monitor'
        WHEN COUNT(*) < 1000 THEN 'Fragmented — fix soon'
        ELSE 'Severely fragmented — fix immediately'
    END                         AS status
FROM sys.dm_db_log_info(NULL)  -- NULL = all databases (SQL 2016+)
GROUP BY database_id
ORDER BY vlf_count DESC;
```
### Fixing VLF Fragmentation
The fix is to **shrink the log to near-zero, then grow it in one large increment** — this creates fewer, larger VLFs.

```sql
-- Step 1: Take log backup (truncates inactive portion, enables shrink)
BACKUP LOG OrdersDB TO DISK = 'D:\Backups\OrdersDB_BeforeVLFFix.trn';

-- Step 2: Shrink the log file (removes inactive VLFs)
USE OrdersDB;
DBCC SHRINKFILE (OrdersDB_log, 1);  -- shrink to 1MB (minimum possible)

-- Step 3: Pre-grow the log to final desired size in ONE operation
-- This creates the ideal number of large VLFs
ALTER DATABASE OrdersDB
MODIFY FILE (
    NAME = OrdersDB_log,
    SIZE = 8192MB    -- 8GB in one shot → 16 VLFs × 512MB each
);
-- Result: 16 large VLFs instead of hundreds of tiny ones ✅

-- Step 4: Verify VLF count
SELECT COUNT(*) AS vlf_count
FROM sys.dm_db_log_info(DB_ID('OrdersDB'));
-- Should now show 16 (or similar small number)
```
### Optimal Log File Sizing
```sql
-- Pre-size log based on expected workload
-- Goal: avoid auto-growth during normal operations

-- Rule of thumb: log should be 10-25% of data file size
-- For busy OLTP: size to hold ~1-2 hours of peak log activity

-- Check log generation rate
SELECT
    DB_NAME(database_id)        AS db_name,
    log_bytes_used / 1024 / 1024 AS log_used_mb,
    log_bytes_used              AS log_bytes_used
FROM sys.dm_db_log_stats(NULL);  -- SQL 2017+

-- Monitor log space usage over time
SELECT
    name,
    log_size_mb = size * 8.0 / 1024,
    log_used_pct = 100 - (100.0 *
        (CAST(FILEPROPERTY(name,'SpaceUsed') AS FLOAT) / size))
FROM sys.database_files
WHERE type_desc = 'LOG';
```
---

## 4. Log Truncation vs Shrink
These are two distinct operations that are frequently confused but do very different things.

### Log Truncation
Truncation **marks inactive VLFs as reusable**. It does NOT reduce the physical file size — it just makes the space available for new log records to overwrite. Truncation is automatic and happens internally.

```
Before truncation:
┌──────────┬──────────┬──────────┬──────────┬──────────┐
│  VLF 1   │  VLF 2   │  VLF 3   │  VLF 4   │  VLF 5   │
│INACTIVE  │INACTIVE  │ ACTIVE   │ ACTIVE   │  FREE    │
│(committed│(committed│(in use)  │(in use)  │          │
│ txns)    │ txns)    │          │          │          │
└──────────┴──────────┴──────────┴──────────┴──────────┘
MinLSN is in VLF 3 — VLFs 1 and 2 can be truncated

After truncation:
┌──────────┬──────────┬──────────┬──────────┬──────────┐
│  VLF 1   │  VLF 2   │  VLF 3   │  VLF 4   │  VLF 5   │
│ REUSABLE │ REUSABLE │ ACTIVE   │ ACTIVE   │  FREE    │
│(space    │(space    │(in use)  │(in use)  │          │
│ freed)   │ freed)   │          │          │          │
└──────────┴──────────┴──────────┴──────────┴──────────┘
File size UNCHANGED — VLFs 1 and 2 just available for reuse
```
### What Triggers Truncation
Truncation requires two conditions:

1. A **checkpoint** has occurred (data pages written to disk)
2. The VLF has no active transactions and is no longer needed
```
Log Truncation Triggers by Recovery Model:
─────────────────────────────────────────────────────────────
Simple Recovery:    After each checkpoint (automatic)
Full Recovery:      After each TRANSACTION LOG BACKUP
Bulk-Logged:        After each log backup
                    (EXCEPT during bulk operations — log holds extents)
─────────────────────────────────────────────────────────────
```
### What Prevents Truncation (Log_Reuse_Wait)
The most important diagnostic — why is the log NOT being truncated?

```sql
-- Find out why log truncation is blocked
SELECT
    name,
    log_reuse_wait,
    log_reuse_wait_desc     -- this is the key column
FROM sys.databases
WHERE name = 'OrdersDB';
```
| log_reuse_wait_desc | Meaning | Fix |
| ----- | ----- | ----- |
|  | Log can be truncated (healthy) | — |
|  | Waiting for checkpoint | <p>Run </p><p> or wait</p> |
|  | Needs a log backup (Full/Bulk-Logged) | Take a log backup |
|  | Open long-running transaction | Identify and resolve long transaction |
|  | Mirror is lagging | Check mirror health |
|  | AG secondary is lagging | Check secondary redo queue |
|  | Log Reader Agent not reading | Check replication agent |
|  | Backup in progress | Wait for backup to complete |
|  | Log scan in progress | Wait |
```sql
-- Find long-running transactions (common cause of log bloat)
SELECT
    es.session_id,
    es.login_name,
    at.transaction_begin_time,
    DATEDIFF(MINUTE, at.transaction_begin_time, GETDATE()) AS minutes_active,
    es.last_request_start_time,
    st.text AS last_sql
FROM sys.dm_exec_sessions es
JOIN sys.dm_tran_session_transactions tst ON es.session_id = tst.session_id
JOIN sys.dm_tran_active_transactions at ON tst.transaction_id = at.transaction_id
CROSS APPLY sys.dm_exec_sql_text(es.most_recent_sql_handle) st
WHERE at.transaction_type = 1   -- read/write transaction
ORDER BY at.transaction_begin_time;
```
### Log Shrink
Shrinking **reduces the physical file size** by removing the empty space at the end of the log file after truncation has freed VLFs.

```sql
-- Step 1: Truncate first (Full recovery model — take log backup)
BACKUP LOG OrdersDB TO DISK = 'D:\Backups\OrdersDB_Log.trn';

-- Step 2: Shrink the log file
USE OrdersDB;
DBCC SHRINKFILE (OrdersDB_log, 256);  -- shrink to 256MB target

-- Check log file name if needed
SELECT name, physical_name FROM sys.database_files WHERE type_desc = 'LOG';
```
### Shrink vs Truncation Comparison
|  | Truncation | Shrink |
| ----- | ----- | ----- |
| Reduces physical file size | ❌ No | ✅ Yes |
| Frees VLFs for reuse | ✅ Yes | ✅ Yes (as part of shrink) |
| Happens automatically | ✅ Yes | ❌ Manual only |
| Can cause VLF fragmentation | ❌ No | ✅ Yes (if done poorly) |
| Recommended for routine use | ✅ Yes | ⚠️ Only when necessary |
### ⚠️ Log Shrink Warnings
```
Shrinking the log is RARELY the right answer for ongoing log growth.
It creates a cycle:
  Shrink → Log grows → Auto-growth → Many small VLFs → Poor performance
  → Shrink again → repeat

Root causes of unexpected log growth:
1. Missing log backups (Full recovery — log never truncated)
2. Long-running open transaction (MinLSN can't advance)
3. Replication / AG secondary lag (log held for delivery)
4. Log file undersized for workload

Fix root causes — don't just shrink repeatedly.
```
---

## 5. WAL (Write-Ahead Logging)
### What Is WAL?
Write-Ahead Logging is the **fundamental protocol** that SQL Server uses to guarantee durability and atomicity. The rule is simple:

>  **Log records describing a change must be written to disk BEFORE the corresponding data page is written to disk.** 

This ensures that after any failure, SQL Server can reconstruct the correct database state by replaying the log — even if data pages were never flushed.

```
Without WAL (dangerous):
1. Modify data page in buffer pool (memory)
2. Write data page to disk
3. Write log record to disk
   → If crash between steps 1 and 2, data page is on disk but log has no record
   → Cannot determine if the change was committed or not ❌

With WAL (safe):
1. Modify data page in buffer pool (memory)
2. Write log record to log buffer (memory)
3. On COMMIT: flush log buffer to disk (.ldf) ← MUST happen before step 4
4. Acknowledge COMMIT to client ✅
5. (Later) Write data page to disk (async — can be deferred)
   → If crash between steps 4 and 5, log has the record
   → Recovery can REDO the change from the log ✅
```
### The Log Flush on Commit
The critical WAL guarantee is that the **log is flushed to disk synchronously on every COMMIT**. SQL Server uses the Log Manager to batch log records and flush when:

1. A transaction commits (`COMMIT TRANSACTION` )
2. The log buffer fills up (60KB threshold)
3. A checkpoint occurs
4. A log backup is taken
```
Log Write Flow:
Transaction modifies data
       │
       ▼
Log record written to LOG BUFFER (in memory — fast)
       │
       ▼ (COMMIT issued)
LOG BUFFER flushed to LOG FILE (.ldf) ← synchronous disk write
       │
       ▼
COMMIT acknowledged to client
       │
       ▼ (asynchronous, later)
Dirty DATA pages written to disk by Checkpoint/Lazy Writer
```
### WRITELOG Wait Type
In high-throughput OLTP systems, the synchronous log flush on commit can become a bottleneck — all transactions must wait for the log write to complete before they can proceed.

```sql
-- Check if log I/O is a bottleneck
SELECT
    wait_type,
    waiting_tasks_count,
    wait_time_ms,
    avg_wait_ms = wait_time_ms / NULLIF(waiting_tasks_count, 0)
FROM sys.dm_os_wait_stats
WHERE wait_type = 'WRITELOG'
ORDER BY wait_time_ms DESC;

-- High WRITELOG wait → log I/O is the bottleneck
-- Solutions:
-- 1. Move log file to faster storage (NVMe SSD, dedicated volume)
-- 2. Reduce commit frequency (batch commits)
-- 3. Use delayed durability (see below)
```
### Delayed Durability (SQL Server 2014+)
For workloads where the WAL guarantee can be relaxed, **Delayed Durability** allows SQL Server to acknowledge commits before the log is flushed to disk. This trades some durability for dramatically higher throughput.

```sql
-- Database-level setting
ALTER DATABASE OrdersDB SET DELAYED_DURABILITY = FORCED;
-- ALL transactions use delayed durability

ALTER DATABASE OrdersDB SET DELAYED_DURABILITY = ALLOWED;
-- Transactions opt in with hint (see below)

ALTER DATABASE OrdersDB SET DELAYED_DURABILITY = DISABLED;
-- Default — full WAL durability for all transactions

-- Per-transaction hint
COMMIT TRANSACTION WITH (DELAYED_DURABILITY = ON);
```
```
Risk of Delayed Durability:
If SQL Server crashes after COMMIT but before the deferred log flush,
committed transactions since the last real flush may be LOST.
Data loss risk = log buffer size (typically < 60KB of transactions)

Acceptable for: high-volume message queues, session state, telemetry
Not acceptable for: financial transactions, inventory, order management
```
### WAL and Storage Configuration
```
WAL performance best practices:
────────────────────────────────────────────────────────────────
1. Dedicated volume for log files — no competition with data I/O
2. Sequential write performance is key — NVMe or fast SSD
3. RAID 1 (mirror) for log — no need for RAID 5/6 (no random reads)
4. Pre-size log to avoid auto-growth during peak periods
5. Separate log from TempDB — both are write-intensive
6. For VMs: use write-through cache or no cache on log volume
   (write-back cache on log = risk of data loss on VM host crash)
────────────────────────────────────────────────────────────────
```
---

## 6. Minimally Logged Operations
### What Is Minimal Logging?
Under the Full recovery model, every row change is **fully logged** — each inserted, updated, or deleted row generates individual log records. This is safe but generates enormous amounts of log for bulk operations.

**Minimally logged** operations write only **extent allocation records** to the log (which extents were allocated) instead of individual row changes. This can reduce log generation by 10–100x for large bulk loads.

```
Fully logged INSERT of 1,000,000 rows:
  1,000,000 × (before image + after image + overhead)
  Typical log size: ~500MB
Minimally logged INSERT of 1,000,000 rows:
  Only extent allocations recorded
  Typical log size: ~5MB
  (100x less log!)
```
### When Minimal Logging Applies
Minimal logging requires **ALL** of the following conditions:

```
Recovery model = SIMPLE or BULK_LOGGED (not FULL)
       AND
Operation is one of the minimally-loggable types:
  - SELECT INTO
  - BULK INSERT / BCP
  - INSERT...SELECT (with specific conditions)
  - CREATE INDEX / ALTER INDEX REBUILD
       AND
Table conditions (for INSERT operations):
  - Table has no non-clustered indexes (or TABLOCK used)
  - Table is empty OR TABLOCK hint is used
```
### Minimally Logged Operations Reference
```sql
-- 1. SELECT INTO (always minimally logged in Bulk-Logged / Simple)
SELECT * INTO dbo.OrdersBackup FROM dbo.Orders;
-- Creates new table and inserts all rows — minimally logged

-- 2. BULK INSERT
ALTER DATABASE OrdersDB SET RECOVERY BULK_LOGGED;  -- switch model

BULK INSERT dbo.Orders
FROM 'C:\Data\Orders.csv'
WITH (
    FIELDTERMINATOR = ',',
    ROWTERMINATOR   = '\n',
    BATCHSIZE       = 50000,         -- commit every 50K rows
    TABLOCK                          -- required for minimal logging
                                     -- if table has non-clustered indexes
);

ALTER DATABASE OrdersDB SET RECOVERY FULL;         -- switch back
BACKUP LOG OrdersDB TO DISK = '...';               -- complete log chain

-- 3. INSERT...SELECT with TABLOCK (Bulk-Logged or Simple)
INSERT INTO dbo.FactSales WITH (TABLOCK)
SELECT * FROM dbo.StagingFactSales;
-- TABLOCK hint enables minimal logging for empty/target table

-- 4. CREATE INDEX (always minimally logged)
CREATE NONCLUSTERED INDEX IX_Orders_Date ON Orders (OrderDate);
-- Index build is minimally logged regardless of recovery model!
-- This is why CREATE INDEX generates much less log than UPDATE

-- 5. ALTER INDEX REBUILD
ALTER INDEX ALL ON Orders REBUILD;
-- Rebuild is minimally logged in Simple and Bulk-Logged
-- In Full recovery model: still generates less log than equivalent UPDATE
```
### INSERT...SELECT Minimal Logging Conditions
This is the trickiest scenario. Minimal logging applies to INSERT...SELECT only when:

```
Target table is empty
  OR (target table has data AND TABLOCK hint is used)
       AND
Recovery model is SIMPLE or BULK_LOGGED
       AND
For heap (no clustered index): always minimal if above conditions met
For clustered index table:
  - Rows must be inserted in clustered key order for minimal logging
  - Otherwise full logging occurs even with TABLOCK

-- Maximally efficient bulk insert pattern:
ALTER DATABASE OrdersDB SET RECOVERY BULK_LOGGED;

INSERT INTO dbo.TargetTable WITH (TABLOCK)  -- lock entire table
SELECT col1, col2, col3
FROM dbo.StagingTable
ORDER BY TargetTable_ClusteredKeyColumn;    -- order by clustered key!

ALTER DATABASE OrdersDB SET RECOVERY FULL;
BACKUP LOG OrdersDB TO DISK = 'D:\Backups\OrdersDB_PostBulk.trn';
```
### Verifying Minimal Logging
```sql
-- Compare log generated for fully vs minimally logged insert
-- Method: capture log records before and after

-- Record current log usage before
DECLARE @LogUsedBefore BIGINT;
SELECT @LogUsedBefore = log_bytes_used
FROM sys.dm_db_log_stats(DB_ID());

-- Do the insert here
INSERT INTO dbo.LargeTable WITH (TABLOCK)
SELECT * FROM dbo.StagingLargeTable;

-- Record log usage after
DECLARE @LogUsedAfter BIGINT;
SELECT @LogUsedAfter = log_bytes_used
FROM sys.dm_db_log_stats(DB_ID());

-- Show how much log was generated
SELECT (@LogUsedAfter - @LogUsedBefore) / 1024 / 1024 AS log_generated_mb;
```
### Bulk-Logged Recovery Model: The Log Backup Caveat
When minimal logging is used, the log backup for that period **includes the actual data pages** (extent bitmaps) that were minimally logged. This means:

```
Bulk-Logged log backup after a large SELECT INTO:
  - Cannot do point-in-time restore during the bulk operation window
  - Log backup may be LARGER than usual (includes extent data)
  - All data loaded is captured — just not at row level
Implication: if you need point-in-time recovery, stay in Full recovery model
             and accept the higher log volume during bulk loads
```
---

## Transaction Log Health Monitoring
```sql
-- Complete log health dashboard
SELECT
    DB_NAME(lf.database_id)             AS database_name,
    -- File size info
    lf.total_log_size_mb,
    lf.active_log_size_mb,
    lf.log_backup_time,
    lf.log_since_last_log_backup_mb,
    lf.log_since_last_checkpoint_mb,
    lf.log_recovery_size_mb,
    -- Truncation status
    d.log_reuse_wait_desc,
    -- VLF count
    vlf.vlf_count,
    -- Log used percentage
    CAST(100.0 * lf.active_log_size_mb /
         NULLIF(lf.total_log_size_mb, 0) AS DECIMAL(5,2)) AS log_used_pct
FROM sys.dm_db_log_stats(NULL) lf              -- SQL 2017+
JOIN sys.databases d ON lf.database_id = d.database_id
JOIN (
    SELECT database_id, COUNT(*) AS vlf_count
    FROM sys.dm_db_log_info(NULL)
    GROUP BY database_id
) vlf ON lf.database_id = vlf.database_id
ORDER BY lf.total_log_size_mb DESC;
```
---

## Transaction Log Quick Reference
| Concept | Key Point | Common Issue |
| ----- | ----- | ----- |
| **Log Structure** | Sequential file of log records in VLFs | Growing without truncation |
| **Log Record** | Before + after image per change | Long transactions hold old log |
| **Crash Recovery** | Analysis → Redo → Undo (ARIES) | Slow if too many VLFs |
| **LSN** | Unique monotonic ID for every log record | Out-of-sequence LSN = broken backup chain |
| **MinLSN** | Oldest log record still needed | If MinLSN can't advance → log can't truncate |
| **VLFs** | Internal log segments; auto-created on growth | Too many VLFs = slow recovery |
| **VLF Fix** | Shrink then pre-grow in one large step | Creates few large VLFs |
| **Log Truncation** | Marks VLFs reusable — no file size change | Blocked by open txn, missing backup, AG lag |
| **Log Shrink** | Reduces physical file size | Causes VLF fragmentation if repeated |
| **WAL** | Log flushed to disk BEFORE commit returns | WRITELOG wait = log I/O bottleneck |
| **Delayed Durability** | Defer log flush for performance | Risk: lose committed txns on crash |
| **Minimal Logging** | Only extents logged, not rows | Requires Bulk-Logged/Simple + TABLOCK |
| **log_reuse_wait** | Reason truncation is blocked | LOG_BACKUP = take a log backup |




<!--- Eraser file: https://app.eraser.io/workspace/cYOIU2a2HNYjQfAqjRRx --->