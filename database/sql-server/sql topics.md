<p><a target="_blank" href="https://app.eraser.io/workspace/8MfW7QxTzcVx7ZiVNEKl" id="edit-in-eraser-github-link"><img alt="Edit in Eraser" src="https://firebasestorage.googleapis.com/v0/b/second-petal-295822.appspot.com/o/images%2Fgithub%2FOpen%20in%20Eraser.svg?alt=media&amp;token=968381c8-a7e7-472a-8ed6-4a6626da5501"></a></p>

## 🗄️ Core SQL & Querying
- Joins (Inner, Left, Right, Full, Cross, Self)
- Subqueries vs CTEs vs Temp Tables vs Table Variables
- Window Functions (ROW_NUMBER, RANK, DENSE_RANK, LEAD, LAG)
- Aggregations and GROUP BY / HAVING
- UNION vs UNION ALL
- EXISTS vs IN vs JOIN
- CASE expressions
- PIVOT and UNPIVOT
- String, Date, and Math functions
---

## 📊 Indexes
- Clustered vs Non-Clustered Index
- Composite Indexes and key column order
- Covering Indexes and INCLUDE columns
- Filtered Indexes
- Index Fragmentation (Rebuild vs Reorganize)
- Missing Index DMVs
- Index Seek vs Index Scan vs Table Scan
- Columnstore Indexes
---

## ⚡ Query Performance & Execution
- Execution Plans (Estimated vs Actual)
- Statistics and how they affect query plans
- Parameter Sniffing
- Query Hints (NOLOCK, RECOMPILE, FORCE ORDER)
- SET STATISTICS IO and SET STATISTICS TIME
- SARGability
- Implicit Conversions and their performance impact
- Optimizer and Plan Cache
---

## 📖 Reads & I/O (the topic we just covered)
- Logical Reads
- Physical Reads
- Read-Ahead Reads
- Buffer Pool and Page lifecycle
- Dirty Pages and Checkpoint
- Lazy Writer
---

## 🔒 Transactions & Locking
- ACID Properties
- Isolation Levels (Read Uncommitted, Read Committed, Repeatable Read, Snapshot, Serializable)
- RCSI (Read Committed Snapshot Isolation)
- Shared, Exclusive, Update, Intent Locks
- Deadlocks — causes, detection, resolution
- Lock Escalation
- BEGIN TRAN, COMMIT, ROLLBACK, SAVEPOINT
- Optimistic vs Pessimistic Concurrency
---

## 🏗️ Database Design & Objects
- Normalization (1NF, 2NF, 3NF, BCNF)
- Denormalization and when to use it
- Primary Key vs Unique Key vs Foreign Key
- Constraints (CHECK, DEFAULT, NOT NULL)
- Views vs Materialized Views (Indexed Views)
- Stored Procedures vs Functions vs Triggers
- Inline TVF vs Multi-statement TVF
- Schemas and their purpose
- Sequences vs Identity columns
---

## 🧠 Stored Procedures & Programmability
- Input/Output Parameters
- Error Handling (TRY/CATCH, THROW, RAISERROR)
- Dynamic SQL and sp_executesql
- Cursors vs Set-based operations
- Temp Tables (#temp) vs Table Variables (@table) vs CTEs
- Recursive CTEs
- MERGE statement
---

## 💾 Memory & Storage
- Buffer Pool
- Memory-Optimized Tables (In-Memory OLTP / Hekaton)
- Data Pages, Extents, and Allocation Units
- Row vs Page vs LOB storage
- Filegroups and data file layout
- TempDB — usage, contention, best practices
- Data compression (Row vs Page compression)
---

## 🔁 High Availability & Disaster Recovery
- Always On Availability Groups
- Database Mirroring (deprecated but still asked)
- Log Shipping
- Replication (Snapshot, Transactional, Merge)
- Failover Clustering
- Backup types (Full, Differential, Transaction Log)
- Recovery Models (Simple, Bulk-Logged, Full)
- RTO vs RPO
---

## 🔐 Security
- Authentication (Windows vs SQL Server)
- Roles (Server roles vs Database roles)
- Row Level Security (RLS)
- Dynamic Data Masking
- Transparent Data Encryption (TDE)
- Column-level Encryption
- GRANT, DENY, REVOKE
- Certificate-based security
---

## 📈 Monitoring & Troubleshooting
- DMVs (Dynamic Management Views) — commonly used ones
- Wait Statistics and wait types
- Blocking and how to detect it
- Deadlock graphs (XML deadlock trace)
- Extended Events vs SQL Profiler (deprecated)
- Query Store
- sys.dm_exec_query_stats, sys.dm_os_wait_stats
- DBCC commands (CHECKDB, FREEPROCCACHE, DROPCLEANBUFFERS)
---

## 🔄 Transaction Log
- How the transaction log works
- Log sequence numbers (LSN)
- Log growth and VLFs (Virtual Log Files)
- Log truncation vs shrink
- WAL (Write-Ahead Logging)
- Minimally logged operations
---

## ☁️ SQL Server in the Cloud / Modern Features
- SQL Server vs Azure SQL Database vs Azure SQL Managed Instance
- Intelligent Query Processing (IQP)
- Adaptive Query Processing
- Accelerated Database Recovery (ADR)
- Batch Mode on Rowstore
- Approximate Query Processing
---

## 📐 Advanced Topics
- Partitioning (Table and Index partitioning)
- Linked Servers
- Distributed Transactions (MSDTC)
- Change Data Capture (CDC) vs Change Tracking
- Full-Text Search
- Graph Tables
- JSON support in SQL Server
- Temporal Tables (System-Versioned)




<!--- Eraser file: https://app.eraser.io/workspace/8MfW7QxTzcVx7ZiVNEKl --->