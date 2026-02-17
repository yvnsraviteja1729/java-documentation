<p><a target="_blank" href="https://app.eraser.io/workspace/ldImojCAvGO4rfxVE9MH" id="edit-in-eraser-github-link"><img alt="Edit in Eraser" src="https://firebasestorage.googleapis.com/v0/b/second-petal-295822.appspot.com/o/images%2Fgithub%2FOpen%20in%20Eraser.svg?alt=media&amp;token=968381c8-a7e7-472a-8ed6-4a6626da5501"></a></p>

# SQL Server Memory & Storage
## Overview
Understanding how SQL Server physically stores and accesses data is fundamental to diagnosing performance problems. Every query ultimately comes down to reading and writing data through a layered system — from logical pages in memory down to physical bytes on disk. Knowing each layer helps you tune at the right level.

```
SQL Query
    │
    ▼
┌──────────────────────────────────────────┐
│         Buffer Pool (Memory)             │  ← 8KB pages cached here
│   [Page][Page][Page][Page][Page]...      │
└──────────────────┬───────────────────────┘
                   │ Miss → fetch from disk
                   ▼
┌──────────────────────────────────────────┐
│         Data Files (.mdf / .ndf)         │  ← Filegroups → Extents → Pages
│   Filegroup → Extents (64KB) → Pages(8KB)│
└──────────────────────────────────────────┘
```
---

## 1. Buffer Pool
### What Is the Buffer Pool?
The Buffer Pool is SQL Server's **primary memory cache** — a region of RAM where SQL Server stores **8KB data pages** that have been read from disk. It is the single most important memory component in SQL Server, and its size directly determines how much I/O SQL Server must perform.

```
Buffer Pool Architecture
────────────────────────────────────────────────────────
┌────────────────────────────────────────────────────┐
│                   Buffer Pool                      │
│                                                    │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐           │
│  │ Clean    │ │ Dirty    │ │ Free     │           │
│  │ Page     │ │ Page     │ │ Page     │  ...      │
│  │(matches  │ │(modified,│ │(available│           │
│  │ disk)    │ │not saved)│ │ for use) │           │
│  └──────────┘ └──────────┘ └──────────┘           │
│                                                    │
│  Managed by: Clock/LRU algorithm                   │
│  Flushed by: Checkpoint + Lazy Writer              │
└────────────────────────────────────────────────────┘
```
### Page States in the Buffer Pool
**Clean pages** — The in-memory page exactly matches what is on disk. Can be evicted (freed) at any time without writing to disk.

**Dirty pages** — The page has been modified in memory but not yet written to disk. Must be written (flushed) before the page frame can be reused.

**Free pages** — Empty frames available for loading new pages from disk.

### How SQL Server Manages Buffer Pool Memory
SQL Server uses a **dynamic memory model** — it grows the buffer pool up to the configured `max server memory` limit, and shrinks it when Windows signals memory pressure via the Memory Notification API.

```sql
-- Configure buffer pool size
EXEC sp_configure 'max server memory (MB)', 16384;  -- 16 GB max
EXEC sp_configure 'min server memory (MB)', 4096;   -- 4 GB min
RECONFIGURE;

-- View current memory usage
SELECT
    physical_memory_in_use_kb / 1024    AS memory_used_mb,
    page_fault_count,
    memory_utilization_percentage
FROM sys.dm_os_process_memory;

-- Buffer pool page details
SELECT
    database_id,
    DB_NAME(database_id)                AS database_name,
    COUNT(*) * 8 / 1024                 AS buffer_pool_mb,
    SUM(CAST(is_modified AS INT)) * 8 / 1024 AS dirty_pages_mb
FROM sys.dm_os_buffer_descriptors
WHERE database_id > 4   -- exclude system databases
GROUP BY database_id
ORDER BY buffer_pool_mb DESC;
```
### Checkpoint
A **checkpoint** is a periodic event where SQL Server writes all **dirty pages** from the buffer pool to disk, and writes a checkpoint record to the transaction log. This limits how much log SQL Server needs to replay during recovery after a crash.

```
Without Checkpoint:          With Regular Checkpoints:
Crash recovery must           Only replay log since
replay ALL log from           last checkpoint
last backup — slow            ── much faster recovery
```
```sql
-- Types of checkpoints
-- Automatic:  triggered when log is ~70% full
-- Manual:     explicit
CHECKPOINT;           -- flush all dirty pages now
CHECKPOINT 10;        -- target recovery time of 10 seconds

-- Indirect checkpoint (per database — recommended SQL 2016+)
ALTER DATABASE MyDB SET TARGET_RECOVERY_TIME = 60 SECONDS;
-- SQL Server checkpoints continuously to maintain this recovery target
```
### Lazy Writer
The **Lazy Writer** is a background process that frees buffer pool memory when SQL Server needs pages for new data. It writes dirty pages to disk and marks frames as free. Unlike Checkpoint (which flushes all dirty pages), Lazy Writer works continuously in small increments based on memory pressure.

```
Lazy Writer triggers when:
- Free page list drops too low
- Memory pressure from OS
- Buffer pool needs pages for incoming data

Lazy Writer actions:
1. Finds LRU (Least Recently Used) pages
2. If dirty → writes to disk first
3. Marks page frame as free
4. Adds to free list for reuse
```
### Buffer Pool Monitoring
```sql
-- Check buffer pool hit rate (should be > 99% in production)
SELECT
    name,
    cntr_value AS buffer_cache_hit_ratio
FROM sys.dm_os_performance_counters
WHERE counter_name = 'Buffer cache hit ratio'
  AND object_name LIKE '%Buffer Manager%';

-- Find which tables consume most buffer pool
SELECT TOP 10
    OBJECT_NAME(i.object_id)  AS table_name,
    i.name                    AS index_name,
    COUNT(*) * 8 / 1024       AS size_mb
FROM sys.dm_os_buffer_descriptors bd
JOIN sys.allocation_units au ON bd.allocation_unit_id = au.allocation_unit_id
JOIN sys.partitions p  ON au.container_id = p.partition_id
JOIN sys.indexes i     ON p.object_id = i.object_id AND p.index_id = i.index_id
WHERE bd.database_id = DB_ID()
GROUP BY i.object_id, i.name
ORDER BY size_mb DESC;
```
---

## 2. Memory-Optimized Tables (In-Memory OLTP / Hekaton)
### What Is In-Memory OLTP?
In-Memory OLTP (codenamed "Hekaton") is a SQL Server feature that stores entire tables **in RAM** using a completely different storage engine — no buffer pool, no disk I/O for reads, no locking. It is designed for extreme OLTP throughput on hot tables.

```
Traditional (Disk-Based):          In-Memory OLTP:
─────────────────────────          ──────────────────────────
Buffer Pool → Disk I/O             Pure RAM — no disk I/O for reads
Locking (shared/exclusive)         Lock-free (optimistic MVCC)
Row-based storage                  Hash index / range index in memory
SQL Server storage engine          Separate Hekaton engine
Plan compilation at runtime        Natively compiled procedures
```
### Key Architecture Concepts
**No locking** — Hekaton uses **multi-version concurrency control (MVCC)**. Every row has begin/end timestamps. Readers see a version consistent with their transaction start time. Writers create new row versions — never block readers.

**Durability options** — Memory-optimized tables can be fully durable (logged to disk) or schema-only durable (data lost on restart — for temp data).

**Natively compiled procedures** — Stored procedures can be compiled directly to machine code (DLL) instead of interpreted T-SQL, achieving the fastest possible execution.

### Creating Memory-Optimized Tables
```sql
-- Step 1: Add memory-optimized filegroup to database
ALTER DATABASE MyDB
ADD FILEGROUP MemOptFG CONTAINS MEMORY_OPTIMIZED_DATA;

ALTER DATABASE MyDB
ADD FILE (
    NAME = 'MemOptData',
    FILENAME = 'C:\Data\MyDB_MemOpt'   -- folder, not file
) TO FILEGROUP MemOptFG;

-- Step 2: Create memory-optimized table
CREATE TABLE dbo.SessionCache (
    SessionID    UNIQUEIDENTIFIER NOT NULL,
    UserID       INT              NOT NULL,
    Data         NVARCHAR(4000)   NULL,
    CreatedAt    DATETIME2        NOT NULL,
    ExpiresAt    DATETIME2        NOT NULL,

    CONSTRAINT PK_SessionCache PRIMARY KEY NONCLUSTERED HASH (SessionID)
        WITH (BUCKET_COUNT = 1000000),      -- hash index

    INDEX IX_SessionCache_UserID NONCLUSTERED (UserID),  -- range index

    CONSTRAINT CHK_SessionCache_Expiry
        CHECK (ExpiresAt > CreatedAt)
)
WITH (
    MEMORY_OPTIMIZED = ON,
    DURABILITY = SCHEMA_AND_DATA   -- fully durable (default)
    -- DURABILITY = SCHEMA_ONLY    -- data lost on restart (for staging/temp)
);
```
### Natively Compiled Stored Procedures
```sql
CREATE PROCEDURE usp_UpsertSession
    @SessionID  UNIQUEIDENTIFIER,
    @UserID     INT,
    @Data       NVARCHAR(4000),
    @ExpiresAt  DATETIME2
WITH NATIVE_COMPILATION, SCHEMABINDING
AS
BEGIN ATOMIC                         -- required for natively compiled procs
WITH (TRANSACTION ISOLATION LEVEL = SNAPSHOT, LANGUAGE = N'English')

    IF EXISTS (SELECT 1 FROM dbo.SessionCache WHERE SessionID = @SessionID)
        UPDATE dbo.SessionCache
        SET Data      = @Data,
            ExpiresAt = @ExpiresAt
        WHERE SessionID = @SessionID;
    ELSE
        INSERT INTO dbo.SessionCache (SessionID, UserID, Data, CreatedAt, ExpiresAt)
        VALUES (@SessionID, @UserID, @Data, SYSUTCDATETIME(), @ExpiresAt);
END;
```
### Hash Index vs Range Index
Memory-optimized tables use special index types:

```sql
-- Hash Index: O(1) point lookups — best for equality (=) only
CONSTRAINT PK PRIMARY KEY NONCLUSTERED HASH (CustomerID)
    WITH (BUCKET_COUNT = 1048576)  -- should be 1-2x expected row count
-- ❌ Cannot do range scans (>, <, BETWEEN)
-- ✅ Perfect for: WHERE CustomerID = @ID

-- Range Index (non-clustered BW-Tree): O(log n) — supports range + equality
INDEX IX_Date NONCLUSTERED (OrderDate)
-- ✅ Supports: WHERE OrderDate BETWEEN @Start AND @End
-- ✅ Supports: WHERE CustomerID = @ID
-- Slower than hash for pure equality
```
### Monitoring In-Memory OLTP
```sql
-- Memory used by in-memory tables
SELECT
    OBJECT_NAME(object_id)           AS TableName,
    memory_allocated_for_table_kb,
    memory_used_by_table_kb
FROM sys.dm_db_xtp_table_memory_stats
WHERE database_id = DB_ID();

-- Check for garbage collection (old row versions)
SELECT
    mem_consumer_name,
    allocated_bytes / 1024 / 1024    AS allocated_mb
FROM sys.dm_xtp_system_memory_consumers;
```
### When to Use In-Memory OLTP
| Good Fit | Poor Fit |
| ----- | ----- |
| Session/state management | Large tables (> available RAM) |
| High-frequency small inserts | Complex analytics/reporting |
| Staging/queue tables | Heavy DDL changes needed |
| Hot lookup tables | Many cross-table transactions |
| Temp tables replacement | Wide rows (many columns) |
---

## 3. Data Pages, Extents, and Allocation Units
### Data Pages
The **8KB page** is SQL Server's fundamental I/O unit. Everything SQL Server reads or writes happens in 8KB page increments — even if you only need 1 row.

```
8KB Page Structure (8,192 bytes total):
┌─────────────────────────────────────────┐
│  Page Header (96 bytes)                 │
│  PageID, PageType, FreeSpace, LSN...    │
├─────────────────────────────────────────┤
│  Row 1 data                             │
│  Row 2 data                             │
│  Row 3 data                             │
│  ...                                    │
│  (up to ~8,060 bytes of row data)       │
├─────────────────────────────────────────┤
│  Row Offset Array (bottom of page)      │
│  [ptr to Row1][ptr to Row2][ptr to Row3]│
└─────────────────────────────────────────┘
```
**Page types:**

| Page Type | Contents |
| ----- | ----- |
| Data (1) | Actual table row data |
| Index (2) | B-Tree index nodes |
| Text/Image (3) | LOB data overflow |
| GAM (8) | Global Allocation Map — tracks extents |
| SGAM (9) | Shared Global Allocation Map |
| IAM (10) | Index Allocation Map — which extents belong to an object |
| PFS (11) | Page Free Space — free space per page |
```sql
-- Inspect a page directly (diagnostic)
DBCC PAGE (MyDB, 1, 245, 3) WITH TABLERESULTS;
-- Args: database, file_id, page_id, print_option (3 = detailed)
```
### Extents
An **extent** is a group of **8 contiguous pages (64KB)**. SQL Server allocates storage in extents to improve I/O efficiency — reading 64KB at once is more efficient than 8 separate 8KB reads.

```
Extent = 8 Pages × 8KB = 64KB

Uniform Extent:              Mixed Extent:
┌──┬──┬──┬──┬──┬──┬──┬──┐   ┌──┬──┬──┬──┬──┬──┬──┬──┐
│T1│T1│T1│T1│T1│T1│T1│T1│   │T1│T2│T1│T3│T2│T1│T4│T2│
└──┴──┴──┴──┴──┴──┴──┴──┘   └──┴──┴──┴──┴──┴──┴──┴──┘
All pages belong to          Pages shared among multiple
one object (large tables)    objects (small/new tables)
```
**Mixed extents** are used for small objects (first 8 pages of a new table). Once an object needs more than 8 pages, it transitions to **uniform extents**.

### Allocation Units
An **allocation unit** is the logical grouping of pages for a specific type of data within a partition. Each table partition has up to 3 allocation units:

```sql
-- View allocation units for a table
SELECT
    au.type_desc        AS allocation_unit_type,
    au.total_pages * 8  AS total_kb,
    au.used_pages * 8   AS used_kb,
    au.data_pages * 8   AS data_kb
FROM sys.allocation_units au
JOIN sys.partitions p ON au.container_id = p.partition_id
WHERE p.object_id = OBJECT_ID('Orders');
```
| Allocation Unit Type | Contains |
| ----- | ----- |
|  | Row data that fits within the 8KB page (≤ 8,060 bytes) |
|  | Variable-length data that pushed a row over 8,060 bytes |
|  | Large object data: TEXT, NTEXT, IMAGE, VARCHAR(MAX), VARBINARY(MAX), XML |
---

## 4. Row vs Page vs LOB Storage
### In-Row Storage
Data that fits within the **8,060 byte row limit** on a single data page. This is the normal, fastest storage path — all data for a row is in one place.

```sql
-- This row easily fits in-row
CREATE TABLE SmallRows (
    ID       INT           NOT NULL,   -- 4 bytes
    Name     VARCHAR(100)  NOT NULL,   -- up to 100 bytes
    Amount   DECIMAL(18,2) NOT NULL,   -- 9 bytes
    IsActive BIT           NOT NULL    -- 1 byte
);
-- Typical row: ~115 bytes → many rows per page → efficient
```
### Row Overflow Storage
When a row's **variable-length columns** push the total row size over 8,060 bytes, SQL Server moves the largest variable-length columns to **overflow pages** in the `ROW_OVERFLOW_DATA` allocation unit. A 24-byte pointer is left in the original row.

```sql
-- This can trigger row overflow
CREATE TABLE WideRows (
    ID        INT              NOT NULL,
    Col1      VARCHAR(4000)    NULL,  -- can be large
    Col2      VARCHAR(4000)    NULL,  -- can be large
    Col3      NVARCHAR(2000)   NULL   -- can be large (2 bytes per char)
);
-- If all cols are near-max, total exceeds 8,060 → overflow kicks in
```
```
Normal row (within 8,060 bytes):
┌─────────────────────────────────┐
│ ID │ Col1_data │ Col2_data │... │  ← all on one data page
└─────────────────────────────────┘

Row overflow (exceeds 8,060 bytes):
┌─────────────────────────────────────┐
│ ID │ Col1 ptr(24B) │ Col2 ptr(24B) │  ← main row on data page
└────────────┬──────────────┬─────────┘
             │              │
             ▼              ▼
      [Col1 data]    [Col2 data]       ← overflow pages
      (separate page)
```
### LOB Storage
**Large Object (LOB)** data types are stored separately in the `LOB_DATA` allocation unit by default — even if they're small enough to fit in-row.

```sql
-- LOB data types
TEXT, NTEXT, IMAGE               -- legacy (deprecated)
VARCHAR(MAX), NVARCHAR(MAX)      -- up to 2GB
VARBINARY(MAX)                   -- up to 2GB
XML                              -- up to 2GB
GEOMETRY, GEOGRAPHY              -- spatial data
```
**LOB storage behavior:**

```sql
-- Default: LOB data stored off-row (even if small)
-- Option: store small LOB values in-row for performance
EXEC sp_tableoption 'Documents', 'large value types out of row', '0';
-- 0 = store in-row if ≤ 8,000 bytes (avoids extra page read for small values)
-- 1 = always store off-row (default)
```
```
LOB Storage Structure:

Main data page:
┌──────────────────────────────────────┐
│ RowID │ Name │ [LOB root pointer]    │  ← 16-byte root pointer
└─────────────────────────┬────────────┘
                          │
                          ▼
              LOB Root Node (B-Tree)
             /          |          \
    [LOB Page 1]  [LOB Page 2]  [LOB Page 3]   ← 8KB LOB pages
    (8,040 bytes) (8,040 bytes) (8,040 bytes)
    (for large values, stored as B-Tree of LOB pages)
```
### Impact on Performance
```sql
-- Avoid SELECT * on tables with LOB columns — forces LOB page reads
SELECT * FROM Documents;  -- ❌ reads all LOB data even if you don't need it

-- Only select LOB when needed
SELECT DocumentID, Title, Summary FROM Documents;        -- ✅ skips LOB pages
SELECT DocumentID, Title, Body    FROM Documents         -- reads LOB only when needed
WHERE DocumentID = @ID;

-- Check LOB reads in execution stats
-- lob logical reads in SET STATISTICS IO output indicates LOB page access
```
---

## 5. Filegroups and Data File Layout
### Database Files
A SQL Server database consists of at minimum two files:

```
MyDatabase
├── Primary Data File (.mdf)     ← required, contains system tables + data
├── Secondary Data File (.ndf)   ← optional, additional data files
└── Transaction Log File (.ldf)  ← required, records all changes
```
```sql
-- View files for current database
SELECT
    name,
    type_desc,
    physical_name,
    size * 8 / 1024      AS size_mb,
    max_size,
    growth,
    is_percent_growth
FROM sys.database_files;
```
### Filegroups
A **filegroup** is a logical container that groups one or more data files together. Tables and indexes are assigned to filegroups, not individual files. SQL Server distributes I/O across all files within a filegroup using a **proportional fill** algorithm.

```sql
-- Default filegroup structure
CREATE DATABASE MyDB
ON PRIMARY (
    NAME = 'MyDB_Primary',
    FILENAME = 'C:\Data\MyDB.mdf',
    SIZE = 512MB,
    MAXSIZE = UNLIMITED,
    FILEGROWTH = 256MB
),
FILEGROUP FG_UserData (
    NAME = 'MyDB_Data1',
    FILENAME = 'D:\Data\MyDB_Data1.ndf',
    SIZE = 2048MB,
    FILEGROWTH = 512MB
),
FILEGROUP FG_Archive (
    NAME = 'MyDB_Archive',
    FILENAME = 'E:\Archive\MyDB_Archive.ndf',
    SIZE = 4096MB,
    FILEGROWTH = 1024MB
)
LOG ON (
    NAME = 'MyDB_Log',
    FILENAME = 'L:\Logs\MyDB.ldf',
    SIZE = 1024MB,
    FILEGROWTH = 256MB
);

-- Set default filegroup
ALTER DATABASE MyDB MODIFY FILEGROUP FG_UserData DEFAULT;

-- Create table on specific filegroup
CREATE TABLE Orders (
    OrderID   INT PRIMARY KEY,
    OrderDate DATETIME
) ON FG_UserData;

-- Move index to different filegroup
CREATE INDEX IX_Orders_Date ON Orders (OrderDate)
ON FG_Archive;
```
### Multiple Files per Filegroup (for Performance)
Adding **multiple equally-sized files** to a filegroup enables SQL Server to perform **parallel I/O** across different physical disks — a common TempDB optimization pattern.

```sql
-- Add multiple files to spread I/O across disks
ALTER DATABASE MyDB ADD FILE (
    NAME = 'MyDB_Data2',
    FILENAME = 'E:\Data\MyDB_Data2.ndf',
    SIZE = 2048MB,
    FILEGROWTH = 512MB
) TO FILEGROUP FG_UserData;

-- Important: all files in a filegroup should be the same size
-- SQL Server uses proportional fill — larger files get more writes
```
### Filegroup Best Practices
```
Drive Layout Recommendation:
─────────────────────────────────────────────────────
C:\  (OS)         → SQL Server binaries only
D:\  (Fast SSD)   → Primary + User data files (.mdf/.ndf)
E:\  (Fast SSD)   → TempDB files (.ndf) — multiple equal files
L:\  (Fast SSD)   → Transaction log files (.ldf)
F:\  (Large HDD)  → Archive filegroup, backups
─────────────────────────────────────────────────────
Separating log from data is the most impactful layout change.
Log writes are sequential; data writes are random.
Different I/O patterns benefit from different storage.
```
```sql
-- Check filegroup free space
SELECT
    fg.name                    AS filegroup_name,
    df.name                    AS file_name,
    df.physical_name,
    df.size * 8 / 1024         AS size_mb,
    (df.size - FILEPROPERTY(df.name, 'SpaceUsed')) * 8 / 1024
                               AS free_mb
FROM sys.filegroups fg
JOIN sys.database_files df ON fg.data_space_id = df.data_space_id;
```
---

## 6. TempDB — Usage, Contention, Best Practices
### What Uses TempDB?
TempDB is the **most shared and most contended** database in SQL Server. Almost every feature that needs temporary storage uses it:

| Feature | TempDB Usage |
| ----- | ----- |
|  | Stored here |
|  | May spill here for large sets |
|  | Spill to disk when memory insufficient |
|  | Spill to disk when memory insufficient |
| Row versioning (RCSI/Snapshot) | Version store lives here |
|  | Work space |
|  | Work space |
|  | Stored here |
| Indexed view materialization | Work space |
```sql
-- Monitor TempDB space usage
SELECT
    SUM(unallocated_extent_page_count) * 8 / 1024     AS free_mb,
    SUM(version_store_reserved_page_count) * 8 / 1024 AS version_store_mb,
    SUM(internal_object_reserved_page_count) * 8 / 1024 AS internal_objects_mb,
    SUM(user_object_reserved_page_count) * 8 / 1024   AS user_objects_mb
FROM sys.dm_db_file_space_usage
WHERE database_id = 2;  -- TempDB is always database_id = 2

-- Find sessions using most TempDB space
SELECT TOP 10
    es.session_id,
    es.login_name,
    es.host_name,
    SUM(tsu.user_objects_alloc_page_count -
        tsu.user_objects_dealloc_page_count) * 8 / 1024 AS user_objects_mb,
    SUM(tsu.internal_objects_alloc_page_count -
        tsu.internal_objects_dealloc_page_count) * 8 / 1024 AS internal_mb
FROM sys.dm_db_task_space_usage tsu
JOIN sys.dm_exec_sessions es ON tsu.session_id = es.session_id
GROUP BY es.session_id, es.login_name, es.host_name
ORDER BY user_objects_mb + internal_mb DESC;
```
### TempDB Contention — GAM/SGAM/PFS Contention
The most common TempDB bottleneck is **allocation page contention**. When many sessions simultaneously create temp tables or spill data, they all compete to update the same system pages (GAM, SGAM, PFS) that track free space.

```sql
-- Detect TempDB allocation contention
SELECT
    wait_type,
    waiting_tasks_count,
    wait_time_ms,
    avg_wait_time_ms = wait_time_ms / NULLIF(waiting_tasks_count, 0)
FROM sys.dm_os_wait_stats
WHERE wait_type IN (
    'PAGELATCH_UP',
    'PAGELATCH_EX',
    'PAGELATCH_SH'
)
ORDER BY wait_time_ms DESC;

-- Check which pages are contended
SELECT
    db_name(database_id)   AS db_name,
    file_id,
    page_id,
    page_type_desc,
    waiting_tasks_count
FROM sys.dm_os_waiting_tasks wt
JOIN sys.dm_exec_requests er ON wt.session_id = er.session_id
WHERE wait_type LIKE 'PAGELATCH%'
  AND database_id = 2;
```
### TempDB Best Practices
**1. Multiple data files (most important)**

```sql
-- Add files equal to logical CPU count (max 8 files)
-- All files MUST be same initial size and growth settings
ALTER DATABASE TempDB ADD FILE (
    NAME = 'tempdev2', FILENAME = 'E:\TempDB\tempdev2.ndf',
    SIZE = 2048MB, FILEGROWTH = 512MB
);
ALTER DATABASE TempDB ADD FILE (
    NAME = 'tempdev3', FILENAME = 'E:\TempDB\tempdev3.ndf',
    SIZE = 2048MB, FILEGROWTH = 512MB
);
ALTER DATABASE TempDB ADD FILE (
    NAME = 'tempdev4', FILENAME = 'E:\TempDB\tempdev4.ndf',
    SIZE = 2048MB, FILEGROWTH = 512MB
);
-- Repeat for desired count (match logical CPU count up to 8)
```
**2. Pre-size TempDB files appropriately**

```sql
-- Set initial size large enough to avoid auto-growth during peaks
-- Auto-growth causes PAGELATCH contention
ALTER DATABASE TempDB
MODIFY FILE (NAME = 'tempdev', SIZE = 4096MB, FILEGROWTH = 512MB);

-- Use fixed growth amount (not percent)
-- Percent growth = unpredictable file sizes = unequal proportional fill
```
**3. Place TempDB on fast dedicated storage**

```
Best: NVMe SSD dedicated volume for TempDB
Good: SSD shared with other databases
Avoid: HDD, shared with log files, shared with OS
```
**4. Avoid unnecessary TempDB usage**

```sql
-- Use memory-optimized table variables for small datasets (SQL 2014+)
-- They avoid TempDB entirely for small row counts

-- Avoid large temp tables inside frequently called procedures
-- They get created and dropped repeatedly → allocation overhead

-- Use OPTION (RECOMPILE) to get accurate memory grants
-- → reduces sort/hash spills to TempDB
SELECT * FROM LargeTable ORDER BY SomeColumn
OPTION (RECOMPILE);  -- fresh row count → better memory grant → less spill
```
**5. SQL Server 2019: TempDB improvements**

SQL Server 2019 introduced **GAM/SGAM latch-free allocation** using optimized metadata (memory-optimized system tables for TempDB metadata), dramatically reducing allocation contention without needing many data files.

```sql
-- Enable in SQL Server 2019 (enabled by default in some editions)
ALTER SERVER CONFIGURATION
SET MEMORY_OPTIMIZED TEMPDB_METADATA = ON;
-- Requires restart; eliminates most GAM/SGAM contention
```
---

## 7. Data Compression (Row vs Page Compression)
### Why Compress?
Data compression reduces the physical size of data on disk and in the buffer pool. This means **more rows fit per page**, which means:

- Fewer pages to read (less I/O)
- More data fits in buffer pool (better cache hit rate)
- Smaller backup files
- **Trade-off:** more CPU to compress/decompress
```
Without compression:  1 page holds 100 rows
With page compression: 1 page holds 300 rows
→ A query reading 1M rows needs 3.3x fewer I/O operations
```
### Row Compression
Row compression stores **fixed-length data types in their actual minimum needed size** rather than their declared size. For example, an `INT` column storing value `5` is stored as 1 byte instead of 4 bytes. It also eliminates NULL and zero-length column storage overhead.

```sql
-- Enable row compression on a table
ALTER TABLE Orders REBUILD WITH (DATA_COMPRESSION = ROW);

-- Enable on a specific index
ALTER INDEX IX_Orders_CustomerID ON Orders
REBUILD WITH (DATA_COMPRESSION = ROW);

-- Create new table with compression
CREATE TABLE Orders (
    OrderID   INT,
    Amount    DECIMAL(18,2)
) WITH (DATA_COMPRESSION = ROW);
```
**What row compression does:**

- Stores integers in minimum bytes needed for their value
- Removes trailing zeros from decimals
- Eliminates fixed-length NULL column storage overhead
- Typical compression ratio: **~20–40%** size reduction
### Page Compression
Page compression includes row compression **plus** two additional techniques: **prefix compression** and **dictionary compression** applied at the page level.

**Prefix compression:** Finds common prefixes within each column on a page and stores them once in a column info structure, replacing repeated prefix bytes with a back-reference.

**Dictionary compression:** Finds common repeated values anywhere on the page and stores them in a dictionary, replacing occurrences with dictionary references.

```sql
-- Enable page compression
ALTER TABLE Orders REBUILD WITH (DATA_COMPRESSION = PAGE);

-- On an index
ALTER INDEX CIX_Orders ON Orders REBUILD WITH (DATA_COMPRESSION = PAGE);

-- On a partitioned table (per partition)
ALTER TABLE OrderHistory REBUILD PARTITION = 1
WITH (DATA_COMPRESSION = PAGE);

ALTER TABLE OrderHistory REBUILD PARTITION = ALL
WITH (DATA_COMPRESSION = PAGE);
```
**Typical compression ratio: ~40–70%** size reduction (depends heavily on data patterns).

### Checking Current Compression
```sql
-- Check compression on all tables and indexes
SELECT
    OBJECT_NAME(p.object_id)   AS table_name,
    i.name                     AS index_name,
    p.partition_number,
    p.data_compression_desc    AS compression_type,
    p.rows,
    au.total_pages * 8 / 1024  AS size_mb
FROM sys.partitions p
JOIN sys.indexes i  ON p.object_id = i.object_id AND p.index_id = i.index_id
JOIN sys.allocation_units au ON p.partition_id = au.container_id
WHERE p.object_id > 100   -- skip system objects
ORDER BY size_mb DESC;
```
### Estimating Compression Savings Before Applying
```sql
-- Estimate compression without actually compressing
EXEC sp_estimate_data_compression_savings
    @schema_name     = 'dbo',
    @object_name     = 'Orders',
    @index_id        = NULL,    -- NULL = all indexes
    @partition_number = NULL,   -- NULL = all partitions
    @data_compression = 'PAGE'; -- 'ROW' or 'PAGE'

-- Output:
-- object_name | schema_name | index_id | size_with_current_compression_setting
-- | size_with_requested_compression_setting | sample_size_with_current | sample_size_with_requested
```
### Columnstore Compression
Columnstore indexes use their own **highly efficient compression** separate from row/page compression. It exploits the fact that columnar data of the same type compresses extremely well.

```sql
-- Columnstore uses its own compression (COLUMNSTORE or COLUMNSTORE_ARCHIVE)
CREATE CLUSTERED COLUMNSTORE INDEX CCI_FactSales ON FactSales
WITH (DATA_COMPRESSION = COLUMNSTORE);           -- default, good balance

-- COLUMNSTORE_ARCHIVE: maximum compression, slower reads
CREATE CLUSTERED COLUMNSTORE INDEX CCI_FactSales ON FactSales
WITH (DATA_COMPRESSION = COLUMNSTORE_ARCHIVE);   -- for cold/archive data
```
### Row vs Page vs Columnstore Compression
| Feature | Row | Page | Columnstore |
| ----- | ----- | ----- | ----- |
| Technique | Minimum byte storage | Prefix + dictionary + row | Delta/RLE/dictionary per column |
| Typical ratio | 20–40% | 40–70% | 5–10x (80–90%) |
| CPU overhead | Low | Moderate | High (but batch mode compensates) |
| Online rebuild | ✅ Enterprise | ✅ Enterprise | ✅ Enterprise |
| Best for | OLTP (slight savings) | Mixed workload | Analytics / data warehouse |
| Works with B-Tree | ✅ Yes | ✅ Yes | N/A (own structure) |
| Works with Columnstore | N/A | N/A | ✅ Yes |
### Compression Decision Guide
```
What is your workload?
├── OLTP, write-heavy, small rows
│     → Try ROW compression — low CPU overhead, moderate savings
│
├── Mixed OLTP + reporting, moderate row sizes
│     → Try PAGE compression — better savings, acceptable CPU cost
│
├── Analytics, large fact tables, mostly reads
│     → Use Columnstore index — 5-10x compression + batch mode
│
├── Archive / historical data, rarely accessed
│     → PAGE or COLUMNSTORE_ARCHIVE — maximum compression
│
└── Unsure?
      → Run sp_estimate_data_compression_savings first
        Compare actual vs estimated — only compress if CPU budget allows
```
### Online vs Offline Rebuild
```sql
-- Offline rebuild (all editions) — table locked during rebuild
ALTER TABLE Orders REBUILD WITH (DATA_COMPRESSION = PAGE);

-- Online rebuild (Enterprise only) — minimal locking
ALTER TABLE Orders REBUILD WITH (
    DATA_COMPRESSION = PAGE,
    ONLINE = ON              -- table accessible during rebuild
);

-- For very large tables — partition-level compression
-- Compress one partition at a time to spread impact
ALTER TABLE OrderHistory REBUILD PARTITION = 3
WITH (DATA_COMPRESSION = PAGE, ONLINE = ON);
```
---

## Memory & Storage Quick Reference
```
Component           Key Facts
─────────────────   ──────────────────────────────────────────────────
Buffer Pool         Primary RAM cache. 8KB pages. Goal: > 99% hit rate
Checkpoint          Flushes dirty pages to disk. Limits crash recovery time
Lazy Writer         Frees buffer pool pages under memory pressure
In-Memory OLTP      Entire table in RAM. Lock-free MVCC. 100x faster for OLTP
Data Page           8KB unit of I/O. ~8,060 bytes usable for row data
Extent              8 pages = 64KB. Unit of allocation
In-Row Storage      Data ≤ 8,060 bytes — stays on data page
Row Overflow        Variable data > 8,060 bytes — overflow allocation unit
LOB Storage         TEXT/VARCHAR(MAX)/XML — LOB allocation unit
Filegroup           Logical file container. Tables assigned to filegroups
TempDB              Most contended DB. Multiple equal files = key tuning
Row Compression     ~20-40% savings. Low CPU. Fixed types → minimum bytes
Page Compression    ~40-70% savings. Moderate CPU. Prefix + dictionary
Columnstore Comp    ~80-90% savings. High CPU. Best for analytics
```
---

## Monitoring Queries Summary
```sql
-- Buffer pool hit rate (target > 99%)
SELECT cntr_value FROM sys.dm_os_performance_counters
WHERE counter_name = 'Buffer cache hit ratio'
  AND object_name LIKE '%Buffer Manager%';

-- TempDB free space
SELECT SUM(unallocated_extent_page_count) * 8 / 1024 AS free_mb
FROM sys.dm_db_file_space_usage WHERE database_id = 2;

-- Current compression on tables
SELECT OBJECT_NAME(object_id), data_compression_desc
FROM sys.partitions WHERE object_id > 100;

-- Estimate page compression savings
EXEC sp_estimate_data_compression_savings 'dbo', 'Orders', NULL, NULL, 'PAGE';

-- Memory-optimized table memory usage
SELECT OBJECT_NAME(object_id), memory_used_by_table_kb
FROM sys.dm_db_xtp_table_memory_stats WHERE database_id = DB_ID();
```




<!--- Eraser file: https://app.eraser.io/workspace/ldImojCAvGO4rfxVE9MH --->