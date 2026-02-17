<p><a target="_blank" href="https://app.eraser.io/workspace/lQzgon6CqIO4dpDK6Kiz" id="edit-in-eraser-github-link"><img alt="Edit in Eraser" src="https://firebasestorage.googleapis.com/v0/b/second-petal-295822.appspot.com/o/images%2Fgithub%2FOpen%20in%20Eraser.svg?alt=media&amp;token=968381c8-a7e7-472a-8ed6-4a6626da5501"></a></p>

# SQL Server High Availability & Disaster Recovery
## Overview
High Availability (HA) and Disaster Recovery (DR) are two distinct but complementary concerns. HA focuses on **minimizing downtime** during planned or unplanned failures (hardware, software, maintenance). DR focuses on **recovering data and services** after a catastrophic event (datacenter failure, corruption, ransomware).

```
HA vs DR at a Glance
─────────────────────────────────────────────────────────────────
High Availability                 Disaster Recovery
─────────────────────────────     ───────────────────────────────
Goal: Minimize downtime           Goal: Recover from catastrophe
Scope: Local / same datacenter    Scope: Remote / different site
Measured by: Uptime %             Measured by: RTO and RPO
Examples: AG, Clustering          Examples: Log Shipping, Backups
Failover: Automatic (seconds)     Failover: Manual (minutes/hours)
─────────────────────────────────────────────────────────────────

The full HA/DR stack for a critical system:
Primary ─► AG Synchronous Replica (HA, same site)
        ─► AG Asynchronous Replica (DR, remote site)
        ─► Log Shipping (additional DR, offsite)
        ─► Full + Differential + Log Backups (point-in-time restore)
```
---

## 1. RTO vs RPO
These two metrics are the **foundation of every HA/DR decision**. Every technology choice involves trading off between them, complexity, and cost.

### Recovery Time Objective (RTO)
**How long can we be down?**

RTO is the maximum acceptable time between a failure occurring and the system being fully operational again. It drives decisions about automation, replica synchronization mode, and failover complexity.

```
Failure occurs at 2:00 PM
RTO = 1 hour
System must be operational by 3:00 PM at the latest

Timeline:
2:00 PM ──► Failure detected
2:05 PM ──► Alert fired, team notified
2:15 PM ──► Decision to failover made
2:20 PM ──► Failover executed
2:30 PM ──► Applications reconnected
2:45 PM ──► System verified operational
            ↑
            45 min total — within 1 hour RTO ✅
```
### Recovery Point Objective (RPO)
**How much data can we afford to lose?**

RPO is the maximum acceptable amount of data loss measured in time. It drives decisions about backup frequency, synchronization mode, and log shipping intervals.

```
Last backup/sync: 8:00 AM
Failure at: 11:30 AM
RPO = 1 hour

Data from 8:00 AM to 11:30 AM = 3.5 hours of data could be lost
This VIOLATES a 1-hour RPO ❌

To meet 1-hour RPO:
→ Take log backups every 15 minutes
→ Use synchronous AG replicas (RPO = 0)
→ Use transaction log shipping every 30 minutes
```
### RTO / RPO by Technology
| Technology | Typical RTO | Typical RPO | Notes |
| ----- | ----- | ----- | ----- |
| Always On AG (Sync) | Seconds (< 30s) | 0 (zero data loss) | Same or nearby site |
| Always On AG (Async) | Minutes (1–5 min) | Seconds to minutes | Remote site DR |
| Failover Clustering | Seconds (< 30s) | 0 (shared storage) | HA only, same site |
| Database Mirroring (Sync) | Seconds | 0 | Deprecated |
| Log Shipping | Minutes to hours | Minutes to hours | Simple DR |
| Backups (Full + Log) | Hours | Minutes (with log backups) | Recovery baseline |
### Defining RTO/RPO for Business Tiers
```
Tier 1 (Mission Critical — e.g. payment processing):
  RTO: < 30 seconds   RPO: 0 (zero data loss)
  Solution: Synchronous AG + Failover Clustering
Tier 2 (Business Critical — e.g. order management):
  RTO: < 5 minutes    RPO: < 1 minute
  Solution: Asynchronous AG with monitoring
Tier 3 (Important — e.g. reporting):
  RTO: < 1 hour       RPO: < 15 minutes
  Solution: Log Shipping + frequent log backups
Tier 4 (Non-critical — e.g. dev/test):
  RTO: < 4 hours      RPO: < 24 hours
  Solution: Daily full backup + differential
```
---

## 2. Always On Availability Groups
Always On Availability Groups (AGs) are the **premier HA/DR solution** in SQL Server 2012 and later. An AG groups one or more databases that fail over together as a unit, with automatic failover and readable secondary replicas.

### Architecture
```
Always On Availability Group: "AG_Production"
──────────────────────────────────────────────────────────────
          Primary Replica                Secondary Replicas
          (Read/Write)                   (Read-only / DR)
┌─────────────────────┐    Redo    ┌──────────────────────┐
│  SQL-PRIMARY        │──────────► │  SQL-SECONDARY-1     │
│  Databases:         │  Log       │  (Sync, same site)   │
│  - OrdersDB         │  Stream    │  HA Replica          │
│  - InventoryDB      │            └──────────────────────┘
│  - CustomerDB       │    Async   ┌──────────────────────┐
│                     │──────────► │  SQL-SECONDARY-2     │
└─────────────────────┘            │  (Async, DR site)    │
         │                         │  DR Replica          │
         │                         └──────────────────────┘
         │
┌────────▼────────────┐
│  AG Listener        │  ← Virtual network name + IP
│  (AG_Listener,      │    Applications connect here
│   192.168.1.100)    │    Automatically routes to primary
└─────────────────────┘
```
### Synchronous vs Asynchronous Commit
**Synchronous Commit** — The primary waits for the secondary to harden the log record to disk before acknowledging the commit to the client. Zero data loss but adds latency (network round-trip per transaction).

```
Synchronous Commit Flow:
Client ──► Primary: COMMIT
Primary ──► Secondary: send log
Secondary: harden to disk
Secondary ──► Primary: ACK
Primary ──► Client: COMMIT confirmed ✅
RPO = 0, RTO = seconds
```
**Asynchronous Commit** — The primary acknowledges the commit immediately without waiting for the secondary. Best for geographically distant replicas where network latency would hurt OLTP performance. Risk: secondary may lag, causing data loss on failover.

```
Asynchronous Commit Flow:
Client ──► Primary: COMMIT
Primary ──► Client: COMMIT confirmed ✅ (immediate)
Primary ──► Secondary: send log (background, best effort)
RPO = seconds to minutes (lag), RTO = minutes
```
### Setting Up an Availability Group
```sql
-- Step 1: Enable Always On on each SQL Server instance
-- (done in SQL Server Configuration Manager or PowerShell)

-- Step 2: Create the AG (run on primary)
CREATE AVAILABILITY GROUP [AG_Production]
WITH (
    AUTOMATED_BACKUP_PREFERENCE = SECONDARY,  -- prefer backups on secondary
    FAILURE_CONDITION_LEVEL = 3,              -- what triggers auto-failover
    HEALTH_CHECK_TIMEOUT = 30000             -- ms before unhealthy
)
FOR DATABASE [OrdersDB], [InventoryDB]        -- databases in this AG
REPLICA ON
    -- Primary replica
    N'SQL-PRIMARY' WITH (
        ENDPOINT_URL          = N'TCP://SQL-PRIMARY:5022',
        AVAILABILITY_MODE     = SYNCHRONOUS_COMMIT,
        FAILOVER_MODE         = AUTOMATIC,
        SECONDARY_ROLE (ALLOW_CONNECTIONS = NO)
    ),
    -- Sync secondary (same site — HA)
    N'SQL-SECONDARY-1' WITH (
        ENDPOINT_URL          = N'TCP://SQL-SECONDARY-1:5022',
        AVAILABILITY_MODE     = SYNCHRONOUS_COMMIT,
        FAILOVER_MODE         = AUTOMATIC,
        SECONDARY_ROLE (ALLOW_CONNECTIONS = READ_ONLY)  -- readable secondary
    ),
    -- Async secondary (remote site — DR)
    N'SQL-SECONDARY-2' WITH (
        ENDPOINT_URL          = N'TCP://SQL-SECONDARY-2:5022',
        AVAILABILITY_MODE     = ASYNCHRONOUS_COMMIT,
        FAILOVER_MODE         = MANUAL,                -- manual DR failover
        SECONDARY_ROLE (ALLOW_CONNECTIONS = READ_ONLY)
    );

-- Step 3: Create the listener
ALTER AVAILABILITY GROUP [AG_Production]
ADD LISTENER N'AG_Listener' (
    WITH IP ((N'192.168.1.100', N'255.255.255.0')),
    PORT = 1433
);
```
### Readable Secondaries
One of the most powerful AG features — route read-only workloads to secondary replicas, offloading the primary.

```sql
-- Application connection string to use readable secondary
-- ApplicationIntent=ReadOnly routes to a secondary replica
"Server=AG_Listener,1433;Database=OrdersDB;
 ApplicationIntent=ReadOnly;IntegratedSecurity=True"

-- Direct query on secondary (must set application intent in connection)
-- Reporting queries, SSRS, ETL sources — all offloaded from primary
SELECT * FROM OrdersDB.dbo.Orders WHERE OrderDate > '2024-01-01';
```
### Monitoring Availability Groups
```sql
-- AG health overview
SELECT
    ag.name                          AS ag_name,
    ar.replica_server_name,
    ar.availability_mode_desc,
    ar.failover_mode_desc,
    ars.role_desc,
    ars.operational_state_desc,
    ars.synchronization_health_desc,
    ars.connected_state_desc
FROM sys.availability_groups ag
JOIN sys.availability_replicas ar     ON ag.group_id = ar.group_id
JOIN sys.dm_hadr_availability_replica_states ars
    ON ar.replica_id = ars.replica_id;

-- Database-level sync status and lag
SELECT
    DB_NAME(drs.database_id)         AS database_name,
    ar.replica_server_name,
    drs.synchronization_state_desc,
    drs.synchronization_health_desc,
    drs.log_send_queue_size,         -- KB not yet sent to secondary
    drs.log_send_rate,               -- KB/sec being sent
    drs.redo_queue_size,             -- KB received but not yet applied
    drs.redo_rate,                   -- KB/sec being applied
    drs.secondary_lag_seconds        -- estimated lag in seconds
FROM sys.dm_hadr_database_replica_states drs
JOIN sys.availability_replicas ar ON drs.replica_id = ar.replica_id;
```
---

## 3. Database Mirroring (Deprecated)
Database Mirroring was the predecessor to Always On AGs, deprecated in SQL Server 2012 and removed in SQL Server 2022. Still commonly asked in interviews because many legacy systems still use it.

### Architecture
```
Database Mirroring
──────────────────────────────────────────────────
┌──────────────┐    Log Stream    ┌──────────────┐
│  Principal   │─────────────────►│  Mirror      │
│  Server      │◄─────────────────│  Server      │
│  (Primary)   │    ACK (sync)    │  (Standby)   │
└──────────────┘                  └──────────────┘
       │                                 │
       └────────────┐  ┌─────────────────┘
                    ▼  ▼
              ┌──────────────┐
              │  Witness     │  ← Optional, enables auto-failover
              │  Server      │    (quorum voting)
              └──────────────┘

Limitations vs Always On AGs:
- One mirror per database (no multiple replicas)
- Mirror is not readable (completely standby)
- Only one database at a time (no failover groups)
- No listener (applications need connection string logic)
```
### Mirroring Modes
```sql
-- High Safety with Automatic Failover (requires Witness)
-- Synchronous: zero data loss, automatic failover
ALTER DATABASE OrdersDB SET PARTNER SAFETY FULL;  -- synchronous
-- With witness → automatic failover on principal failure

-- High Performance (no Witness)
-- Asynchronous: minimal performance impact, manual failover
ALTER DATABASE OrdersDB SET PARTNER SAFETY OFF;   -- asynchronous

-- High Safety without Automatic Failover
-- Synchronous, no witness → manual failover only
```
### Key Mirroring Concepts
| Concept | Detail |
| ----- | ----- |
| Principal | Active read/write database |
| Mirror | Warm standby — not readable, not accessible |
| Witness | Optional SQL Express instance for quorum/auto-failover |
| Endpoint | TCP endpoint used for log stream communication |
| Operating Mode | High Safety (sync), High Performance (async) |
### Why It Was Deprecated
- Single database scope (no group failover)
- Mirror not readable (wasted resources)
- No listener support natively
- No support for more than one standby
- Always On AGs supersede it in every way
```sql
-- Checking mirroring status (legacy systems)
SELECT
    DB_NAME(database_id)     AS database_name,
    mirroring_state_desc,
    mirroring_role_desc,
    mirroring_safety_level_desc,
    mirroring_partner_name,
    mirroring_witness_name,
    mirroring_witness_state_desc
FROM sys.database_mirroring
WHERE mirroring_state IS NOT NULL;
```
---

## 4. Log Shipping
Log Shipping is the **simplest HA/DR technology** — it periodically copies transaction log backups from a primary server to one or more secondary servers and restores them automatically. It's essentially automated log backup + restore.

### Architecture
```
Log Shipping Flow
────────────────────────────────────────────────────────────
PRIMARY SERVER                    SECONDARY SERVER(S)
─────────────────                 ─────────────────────────
OrdersDB (Read/Write)             OrdersDB_DR (Standby)
      │                                   │
      │ 1. SQL Agent Job:                 │
      │    Backup Transaction Log         │
      │    every 15 minutes               │
      │    → \\FileShare\LogBackups\      │
      │                                   │
      │                      2. SQL Agent Job:
      │                         Copy log backup files
      │                         from share to local
      │                                   │
      │                      3. SQL Agent Job:
      │                         Restore log backups
      │                         (NORECOVERY or STANDBY)
      │                                   │
      ▼                                   ▼
 ┌──────────┐                     ┌──────────────┐
 │ Log      │                     │ Monitor      │  ← Optional: alerts
 │ Backups  │                     │ Server       │    on lag threshold
 └──────────┘                     └──────────────┘
```
### Configuring Log Shipping
```sql
-- Primary: take log backup
BACKUP LOG OrdersDB
TO DISK = '\\FileShare\LogShipping\OrdersDB_' +
          CONVERT(VARCHAR, GETDATE(), 112) + '.trn'
WITH COMPRESSION, STATS = 10;

-- Secondary: restore with NORECOVERY (database stays in restoring state)
-- OR STANDBY (database is readable but in warm standby mode)

-- Restore with STANDBY (readable secondary — most useful)
RESTORE LOG OrdersDB
FROM DISK = '\\FileShare\LogShipping\OrdersDB_20240101.trn'
WITH STANDBY = 'C:\Standby\OrdersDB_UNDO.bak',
     STATS = 10;
-- Database is readable between log restores ✅

-- Restore with NORECOVERY (not readable, lower overhead)
RESTORE LOG OrdersDB
FROM DISK = '\\FileShare\LogShipping\OrdersDB_20240101.trn'
WITH NORECOVERY;

-- Failover: bring secondary online
RESTORE DATABASE OrdersDB WITH RECOVERY;
-- Database is now fully online — becomes the new primary
```
### Log Shipping Restore Modes
| Mode | Secondary Readable? | Notes |
| ----- | ----- | ----- |
|  | ❌ No | Database in "Restoring" state |
|  | ✅ Yes (between restores) | Requires undo file, brief disconnect during each restore |
### Monitoring Log Shipping
```sql
-- Check log shipping status
SELECT
    primary_server,
    primary_database,
    backup_threshold,           -- minutes before alert fires
    threshold_alert_enabled,
    last_backup_file,
    last_backup_date,
    last_copied_file,
    last_copied_date,
    last_restored_file,
    last_restored_date,
    last_restored_latency       -- minutes of lag
FROM msdb.dbo.log_shipping_monitor_primary;

-- Secondary status
SELECT
    secondary_server,
    secondary_database,
    last_restored_file,
    last_restored_date,
    restore_threshold,          -- max minutes before alert
    last_restore_latency
FROM msdb.dbo.log_shipping_monitor_secondary;
```
### Log Shipping vs Always On AGs
| Feature | Log Shipping | Always On AG |
| ----- | ----- | ----- |
| RPO | Minutes (backup interval) | Seconds (sync) or near-zero (async) |
| RTO | Minutes to hours | Seconds (automatic) |
| Readable secondary | ✅ With STANDBY mode | ✅ Built-in |
| Multiple secondaries | ✅ Yes | ✅ Yes (up to 8) |
| Automatic failover | ❌ Manual only | ✅ Yes (sync mode) |
| Cross-version support | ✅ Yes (different versions) | ❌ Same version required |
| Complexity | Low | High |
| License requirement | All editions | Enterprise (most features) |
---

## 5. Replication
Replication copies and distributes data from one database (**Publisher**) to one or more databases (**Subscribers**). Unlike HA technologies, replication is often used for **data distribution, scale-out reads, and ETL** rather than failover.

### Replication Components
```
Replication Architecture
──────────────────────────────────────────────────────
┌──────────────┐    Articles    ┌──────────────────┐
│  Publisher   │───────────────►│   Distributor    │
│  (Source DB) │                │   (Stores &      │
│              │                │   forwards data) │
└──────────────┘                └────────┬─────────┘
                                         │
                    ┌────────────────────┼──────────────────┐
                    ▼                    ▼                   ▼
             ┌────────────┐      ┌────────────┐    ┌────────────┐
             │ Subscriber │      │ Subscriber │    │ Subscriber │
             │    #1      │      │    #2      │    │    #3      │
             └────────────┘      └────────────┘    └────────────┘

Components:
- Publication: logical grouping of articles to replicate
- Article: a table, stored procedure, or view being replicated
- Distributor: stores replicated transactions (distribution database)
- Subscription: Pull (subscriber requests) or Push (distributor sends)
```
### Snapshot Replication
Delivers a **complete snapshot** of all published data to subscribers at a scheduled point in time. The entire dataset is re-copied each time.

```
Snapshot Replication Flow:
1. Snapshot Agent runs (e.g., nightly)
2. Locks tables briefly, generates BCP files
3. Snapshot Agent stores files in snapshot folder
4. Distribution Agent applies snapshot to subscribers
5. Subscriber receives FULL copy of data

Best for:
- Small, infrequently changing datasets
- Lookup/reference tables (product catalog, zip codes)
- Initial synchronization for other replication types
- Subscribers that don't need near-real-time data
```
```sql
-- Create snapshot publication
EXEC sp_addpublication
    @publication         = N'Pub_ProductCatalog',
    @repl_freq           = N'snapshot',       -- snapshot type
    @sync_method         = N'bcp native',
    @description         = N'Product catalog snapshot';

-- Add article (table)
EXEC sp_addarticle
    @publication = N'Pub_ProductCatalog',
    @article     = N'Products',
    @source_table = N'Products',
    @type        = N'logbased';               -- log-based change tracking
```
### Transactional Replication
Captures **individual transactions** from the publisher's transaction log and delivers them to subscribers in near real-time. The Log Reader Agent continuously reads the log and forwards changes to the distributor.

```
Transactional Replication Flow:
1. Transaction committed on Publisher
2. Log Reader Agent reads changes from transaction log
3. Changes stored in Distribution database
4. Distribution Agent applies changes to Subscriber(s)
   (nearly real-time — typical latency: seconds)

Best for:
- Near real-time data distribution
- Scale-out read replicas (reporting, ETL sources)
- Heterogeneous publishing (to Oracle, MySQL)
- One-way data distribution (publisher is source of truth)
```
```sql
-- Create transactional publication
EXEC sp_addpublication
    @publication  = N'Pub_Orders_Transactional',
    @repl_freq    = N'continuous',    -- continuous change capture
    @sync_method  = N'concurrent',
    @allow_push   = N'true',
    @allow_pull   = N'true';

-- Monitor replication latency using tracer tokens
EXEC sp_posttracertoken
    @publication = N'Pub_Orders_Transactional';

-- Check latency
SELECT TOP 5
    tracer_id,
    publisher_commit,
    distributor_commit,
    subscriber_commit,
    DATEDIFF(SECOND, publisher_commit, subscriber_commit) AS total_latency_sec
FROM MStracer_tokens mt
JOIN MStracer_history mth ON mt.tracer_id = mth.parent_tracer_id
ORDER BY publisher_commit DESC;
```
### Merge Replication
Both publisher and subscribers can make changes, and those changes are **merged and synchronized** bidirectionally. Conflicts are detected and resolved based on rules you define.

```
Merge Replication Flow:
Publisher ◄──► Synchronize ◄──► Subscriber 1 (makes changes)
              (Merge Agent)
                  │
                  ▼
             Subscriber 2 (also makes changes)
             Conflict resolution applied

Best for:
- Mobile/disconnected users who sync periodically
- Multi-site data entry with later synchronization
- Applications where each location needs local read/write
```
```sql
-- Conflict resolution options for merge replication
-- Winner of conflict is determined by:
-- 1. Priority-based (higher priority wins)
-- 2. First change wins
-- 3. Last change wins
-- 4. Custom resolver (stored procedure)

EXEC sp_addmergepublication
    @publication         = N'Pub_FieldData_Merge',
    @sync_mode           = N'native',
    @conflict_policy     = N'pub wins',    -- publisher always wins conflicts
    @description         = N'Field data merge';
```
### Replication Types Comparison
| Feature | Snapshot | Transactional | Merge |
| ----- | ----- | ----- | ----- |
| Data flow | Publisher → Subscriber | Publisher → Subscriber | Bidirectional |
| Latency | Hours (scheduled) | Seconds | Minutes to hours |
| Subscriber writes | ❌ Read-only | ❌ Read-only | ✅ Yes |
| Conflict handling | N/A | N/A | ✅ Required |
| Overhead on publisher | Low (scheduled) | Moderate (log reader) | High (triggers) |
| Best for | Reference data | Reporting scale-out | Mobile/disconnected |
---

## 6. Failover Clustering (FCI)
A SQL Server Failover Cluster Instance (FCI) is a **Windows Server Failover Cluster (WSFC)** configuration where SQL Server is installed as a clustered resource. Multiple nodes share a single set of data — there is no data replication, only a single instance that moves between nodes on failure.

### Architecture
```
Failover Cluster Instance (FCI)
──────────────────────────────────────────────────────────────
┌───────────────┐    ┌───────────────┐
│   Node 1      │    │   Node 2      │
│ (Active)      │    │ (Passive)     │
│               │    │               │
│  SQL Server ◄─┼────┼─ Heartbeat   │
│  Instance     │    │               │
└───────┬───────┘    └───────────────┘
        │ Shared Storage
        ▼
┌───────────────────────────────┐
│  Shared Storage (SAN / S2D)   │  ← Both nodes have access
│  - Data files (.mdf/.ndf)     │    but only active node uses it
│  - Log files (.ldf)           │
│  - TempDB                     │
└───────────────────────────────┘
        │
        ▼
┌───────────────────────────────┐
│  Virtual Network Name         │  ← Applications connect here
│  SQL-CLUSTER (10.0.0.100)     │    automatically follows active node
└───────────────────────────────┘
```
### How FCI Failover Works
```
Normal Operation:
Node 1 (Active) ── runs SQL Server ── serves requests
Node 2 (Passive) ── idle, monitors Node 1 heartbeat

Failure Event:
Node 1 stops responding (hardware crash, OS failure)
WSFC detects heartbeat loss after timeout
WSFC moves SQL Server resources to Node 2:
  1. Takes ownership of shared storage
  2. Starts SQL Server service
  3. Brings VNN (Virtual Network Name) online
  4. SQL Server runs crash recovery on Node 2

RTO: 30 seconds to 2 minutes (depends on recovery time)
RPO: 0 (shared storage — no data loss, data was never copied)
```
### FCI vs Always On AG
| Feature | FCI | Always On AG |
| ----- | ----- | ----- |
| Data storage | Single shared copy | Separate copy per replica |
| RPO | 0 (no data movement) | 0 (sync) or seconds (async) |
| RTO | 1–3 minutes (recovery) | < 30 seconds (no recovery needed) |
| Readable secondary | ❌ No | ✅ Yes |
| Multiple databases failover | ✅ Yes (whole instance) | ✅ Yes (per AG group) |
| Storage cost | High (SAN/S2D required) | Lower (local storage per node) |
| Protects against | Node/OS failure | Broader (DB corruption, datacenter) |
| License | All editions | Enterprise (full features) |
### FCI + AG (Combined Architecture)
In enterprise environments, FCI and AG are often **combined** for maximum protection:

```
AG with FCI nodes:
┌──────────────────────┐     ┌──────────────────────┐
│  FCI-1 (2 nodes)     │─────│  FCI-2 (2 nodes)     │
│  Active/Passive      │ AG  │  Active/Passive       │
│  Primary Replica     │     │  Secondary Replica    │
└──────────────────────┘     └──────────────────────┘
FCI handles node failure       AG handles site/instance failure
within a site                  across sites
```
### Key FCI Configuration
```sql
-- Check FCI nodes
SELECT
    NodeName,
    status_description,
    is_current_owner
FROM sys.dm_os_cluster_nodes;

-- Check cluster resources
SELECT
    cluster_name,
    quorum_type_desc,
    quorum_state_desc
FROM sys.dm_hadr_cluster;
```
---

## 7. Backup Types
Backups are the **last line of defense** in any DR strategy. Even with AGs and clustering, backups protect against logical corruption, accidental deletion, ransomware, and long-term retention requirements.

### Full Backup
Captures **every page** of the database that contains data. This is the baseline from which all other restores start. Every restore chain begins with a full backup.

```sql
-- Basic full backup
BACKUP DATABASE OrdersDB
TO DISK = 'D:\Backups\OrdersDB_Full_20240101.bak'
WITH
    COMPRESSION,                         -- compress backup file
    CHECKSUM,                            -- validate pages during backup
    STATS = 10,                          -- report progress every 10%
    DESCRIPTION = 'Weekly full backup';

-- Full backup to multiple files (parallel striped — faster)
BACKUP DATABASE OrdersDB
TO
    DISK = 'D:\Backups\OrdersDB_Full_01.bak',
    DISK = 'E:\Backups\OrdersDB_Full_02.bak',
    DISK = 'F:\Backups\OrdersDB_Full_03.bak'
WITH COMPRESSION, CHECKSUM, STATS = 10;

-- Copy-only backup (doesn't break differential chain)
BACKUP DATABASE OrdersDB
TO DISK = 'D:\Backups\OrdersDB_CopyOnly.bak'
WITH COPY_ONLY, COMPRESSION;
-- Use COPY_ONLY for ad-hoc backups that shouldn't affect regular schedule
```
### Differential Backup
Captures only **pages changed since the last full backup**. Faster to take and smaller than a full backup. Restore requires: Full + most recent Differential.

```sql
-- Differential backup
BACKUP DATABASE OrdersDB
TO DISK = 'D:\Backups\OrdersDB_Diff_20240101_1800.bak'
WITH DIFFERENTIAL, COMPRESSION, CHECKSUM, STATS = 10;

-- Differential backups grow over time since the last full
-- Monday full:        2 GB
-- Tuesday diff:       400 MB (changes since Monday full)
-- Wednesday diff:     700 MB (changes since Monday full — larger)
-- Thursday diff:      950 MB (changes since Monday full — larger still)
-- Friday full:        2 GB  (resets differential base)
-- Friday diff:        100 MB (changes since new Friday full — small again)
```
### Transaction Log Backup
Captures the **active transaction log** since the last log backup. Enables point-in-time restore and keeps the log file from growing unboundedly (truncates inactive log after backup). Only available in Full and Bulk-Logged recovery models.

```sql
-- Transaction log backup
BACKUP LOG OrdersDB
TO DISK = 'D:\Backups\OrdersDB_Log_20240101_1415.trn'
WITH COMPRESSION, CHECKSUM, STATS = 10;

-- Tail-log backup (before restoring from failure)
-- Captures the log records not yet backed up — minimizes data loss
BACKUP LOG OrdersDB
TO DISK = 'D:\Backups\OrdersDB_TailLog.trn'
WITH NO_TRUNCATE,           -- backup even if database is damaged
     NORECOVERY,            -- leave database in restoring state
     COMPRESSION;
```
### Restore Sequences
```sql
-- Restore from Full + Differential + Log backups

-- Step 1: Restore full backup (NORECOVERY = more to come)
RESTORE DATABASE OrdersDB
FROM DISK = 'D:\Backups\OrdersDB_Full_20240101.bak'
WITH NORECOVERY, STATS = 10;

-- Step 2: Restore most recent differential (NORECOVERY)
RESTORE DATABASE OrdersDB
FROM DISK = 'D:\Backups\OrdersDB_Diff_20240104.bak'
WITH NORECOVERY, STATS = 10;

-- Step 3: Restore log backups in sequence (NORECOVERY)
RESTORE LOG OrdersDB FROM DISK = 'D:\Backups\OrdersDB_Log_1.trn' WITH NORECOVERY;
RESTORE LOG OrdersDB FROM DISK = 'D:\Backups\OrdersDB_Log_2.trn' WITH NORECOVERY;
RESTORE LOG OrdersDB FROM DISK = 'D:\Backups\OrdersDB_Log_3.trn' WITH NORECOVERY;

-- Step 4: Point-in-time restore (optional — restore to specific moment)
RESTORE LOG OrdersDB
FROM DISK = 'D:\Backups\OrdersDB_Log_4.trn'
WITH NORECOVERY,
     STOPAT = '2024-01-04 14:23:00';  -- stop at exact moment before incident

-- Step 5: Bring database online (RECOVERY = no more to apply)
RESTORE DATABASE OrdersDB WITH RECOVERY;
```
### Backup Verification
```sql
-- Verify backup integrity without restoring
RESTORE VERIFYONLY
FROM DISK = 'D:\Backups\OrdersDB_Full_20240101.bak'
WITH CHECKSUM;

-- Check backup history
SELECT TOP 20
    bs.database_name,
    bs.type                                   AS backup_type,
    bs.backup_start_date,
    bs.backup_finish_date,
    DATEDIFF(SECOND, bs.backup_start_date,
             bs.backup_finish_date)           AS duration_sec,
    bs.backup_size / 1024 / 1024              AS backup_mb,
    bs.compressed_backup_size / 1024 / 1024  AS compressed_mb,
    bmf.physical_device_name
FROM msdb.dbo.backupset bs
JOIN msdb.dbo.backupmediafamily bmf ON bs.media_set_id = bmf.media_set_id
WHERE bs.database_name = 'OrdersDB'
ORDER BY bs.backup_start_date DESC;
```
### Backup Types Summary
| Type | Contents | Restore Requires | When to Use |
| ----- | ----- | ----- | ----- |
| Full | All data pages | Nothing else (baseline) | Weekly or nightly |
| Differential | Changed pages since last Full | Full + this diff | Daily |
| Transaction Log | Log since last log backup | Full + diff + log chain | Every 15 min |
| Copy-Only Full | All data (no chain impact) | Nothing else | Ad-hoc |
| File/Filegroup | Specific files | Coordinated log backups | Very large DBs |
| Partial | Read/write filegroups only | Full partial + logs | Large DBs with read-only FGs |
---

## 8. Recovery Models
The recovery model controls **how SQL Server manages the transaction log** — specifically what gets logged, how long log records are retained, and what backup and restore options are available.

### Simple Recovery Model
The transaction log is **automatically truncated** after each checkpoint. Log space is reused aggressively. No transaction log backups possible — you can only restore to the last full or differential backup.

```sql
ALTER DATABASE OrdersDB SET RECOVERY SIMPLE;

-- What's possible:
-- ✅ Full backups
-- ✅ Differential backups
-- ❌ Transaction log backups (log auto-truncated)
-- ❌ Point-in-time restore

-- Log file behavior:
-- After checkpoint → inactive log truncated automatically
-- Log file stays small → minimal disk usage
```
```
Simple Recovery Timeline:
Full    Diff    Diff    Full    Diff
├───────┼───────┼───────┼───────┼──────►
       ↑       ↑
    Can only restore to these points
    Any changes between backups are LOST on failure
```
**Best for:** Development, test, reporting databases, data warehouses where bulk loads run nightly and individual transactions aren't critical.

### Full Recovery Model
**Every transaction is fully logged** and log records are retained until a log backup is taken. Enables point-in-time restore. Required for log shipping and Always On AGs.

```sql
ALTER DATABASE OrdersDB SET RECOVERY FULL;

-- What's possible:
-- ✅ Full backups
-- ✅ Differential backups
-- ✅ Transaction log backups
-- ✅ Point-in-time restore
-- ✅ Log Shipping, Always On AG

-- IMPORTANT: Log file grows until log backup is taken
-- Must schedule regular log backups or log fills up
```
```
Full Recovery Timeline:
Full         Diff    L  L  L  Diff    L  L  L  L
├────────────┼───────┼──┼──┼──┼───────┼──┼──┼──┼──►
                                                ↑
                              Can restore to any point in time
```
**Best for:** All production OLTP databases, any database where data loss is unacceptable, databases participating in log shipping or AGs.

### Bulk-Logged Recovery Model
A **hybrid model** — most operations are fully logged, but certain bulk operations are **minimally logged** (only extent allocations, not every row). This dramatically speeds up bulk operations but at the cost of losing point-in-time restore capability during bulk operations.

```sql
ALTER DATABASE OrdersDB SET RECOVERY BULK_LOGGED;

-- Minimally logged operations:
-- ✅ BULK INSERT, BCP
-- ✅ SELECT INTO
-- ✅ CREATE INDEX
-- ✅ INSERT...SELECT (with some conditions)

-- Behavior:
-- Log backups still work
-- But log backup taken DURING bulk operation includes all changed extents
-- (may be larger than expected)
-- Cannot do point-in-time restore if bulk operation is in the log backup
```
**Typical pattern:** Switch to Bulk-Logged for the duration of a large ETL load, then switch back to Full.

```sql
-- Pattern: Bulk-Logged for ETL window
ALTER DATABASE OrdersDB SET RECOVERY BULK_LOGGED;

BULK INSERT FactSales
FROM 'C:\Data\DailySales.csv'
WITH (FIELDTERMINATOR = ',', ROWTERMINATOR = '\n', BATCHSIZE = 10000);

-- Switch back to Full immediately after
ALTER DATABASE OrdersDB SET RECOVERY FULL;

-- Take log backup to complete the log chain
BACKUP LOG OrdersDB TO DISK = 'D:\Backups\OrdersDB_AfterBulk.trn'
WITH COMPRESSION;
```
### Recovery Model Comparison
| Feature | Simple | Full | Bulk-Logged |
| ----- | ----- | ----- | ----- |
| Log backups | ❌ No | ✅ Yes | ✅ Yes |
| Point-in-time restore | ❌ No | ✅ Yes | ⚠️ Partial |
| Log auto-truncation | ✅ Yes (checkpoint) | ❌ No | ❌ No |
| Log file size | Small | Grows until backup | Grows until backup |
| Bulk operation logging | Full | Full | Minimal |
| Log Shipping support | ❌ No | ✅ Yes | ✅ Yes |
| Always On AG support | ❌ No | ✅ Yes | ✅ Yes |
| Best for | Dev/Test/DW | Production OLTP | ETL/Bulk loads |
---

## HA/DR Technology Selection Guide
```
What are your requirements?
│
├── RPO = 0, RTO < 30 seconds, same datacenter?
│     └── Always On AG (Synchronous) + Failover Clustering
│
├── RPO near-zero, RTO < 5 min, cross-datacenter?
│     └── Always On AG (Asynchronous, remote replica)
│
├── RPO < 15 min, RTO < 1 hour, simple setup?
│     └── Log Shipping (STANDBY mode for readable secondary)
│
├── Need scale-out reads (reporting, ETL) from secondary?
│     └── Always On AG (readable secondaries)
│        OR Log Shipping with STANDBY mode
│        OR Transactional Replication
│
├── Distribute data to many subscribers (reporting servers)?
│     └── Transactional Replication
│
├── Mobile/disconnected scenario, bidirectional sync?
│     └── Merge Replication
│
├── Reference/lookup tables to many subscribers?
│     └── Snapshot Replication
│
└── Protect against ransomware/corruption/accidental delete?
      └── Backups (Full + Differential + Log) — ALWAYS in addition to above
          Offsite copy essential (3-2-1 backup rule)
```
---

## Quick Reference
| Technology | RPO | RTO | Direction | Readable Secondary | Auto-Failover |
| ----- | ----- | ----- | ----- | ----- | ----- |
| **Always On AG (Sync)** | 0 | Seconds | One-way | ✅ Yes | ✅ Yes |
| **Always On AG (Async)** | Seconds | Minutes | One-way | ✅ Yes | ❌ Manual |
| **FCI** | 0 | 1–3 min | N/A (shared) | ❌ No | ✅ Yes |
| **DB Mirroring (Sync)** | 0 | Seconds | One-way | ❌ No | ✅ With Witness |
| **Log Shipping** | Minutes | Minutes–Hours | One-way | ✅ STANDBY mode | ❌ Manual |
| **Snapshot Replication** | Hours | Minutes | One-way | ✅ Yes | ❌ N/A |
| **Transactional Replication** | Seconds | Minutes | One-way | ✅ Yes | ❌ N/A |
| **Merge Replication** | Minutes | Minutes | Bidirectional | ✅ Yes | ❌ N/A |
| **Full Backup** | Hours | Hours | N/A | N/A | ❌ Manual |
| **Full + Log Backups** | Minutes | Hours | N/A | N/A | ❌ Manual |




<!--- Eraser file: https://app.eraser.io/workspace/lQzgon6CqIO4dpDK6Kiz --->