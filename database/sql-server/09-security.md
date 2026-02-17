<p><a target="_blank" href="https://app.eraser.io/workspace/63gHFhnHXj5u7JccNU20" id="edit-in-eraser-github-link"><img alt="Edit in Eraser" src="https://firebasestorage.googleapis.com/v0/b/second-petal-295822.appspot.com/o/images%2Fgithub%2FOpen%20in%20Eraser.svg?alt=media&amp;token=968381c8-a7e7-472a-8ed6-4a6626da5501"></a></p>

# SQL Server Security
## Overview
SQL Server security is built in layers — each layer provides a different kind of protection, and a well-secured SQL Server uses all of them together. The principle of **least privilege** runs through every layer: grant the minimum permissions required, nothing more.

```
SQL Server Security Layers
──────────────────────────────────────────────────────────────
Layer 1: Network          Firewall, encrypted connections (TLS)
Layer 2: Authentication   Who are you? (Windows / SQL Login)
Layer 3: Authorization    What can you do? (Roles, GRANT/DENY)
Layer 4: Object Security  What objects can you access?
Layer 5: Row Security     Which rows can you see? (RLS)
Layer 6: Column Security  Which column values can you see? (DDM)
Layer 7: Encryption       Can data be read if storage is stolen? (TDE)
Layer 8: Key Security     Who manages encryption keys? (Certificates)
──────────────────────────────────────────────────────────────

Defense in Depth: compromise of one layer does not expose data
if other layers are properly configured.
```
---

## 1. Authentication (Windows vs SQL Server)
Authentication is the process of **verifying identity** — proving that a connection is who it claims to be. SQL Server supports two authentication modes.

### Windows Authentication (Integrated Security)
The user's Windows identity (Active Directory) is passed to SQL Server through Kerberos or NTLM. SQL Server trusts Windows to have already verified the password — no separate SQL Server password needed.

```
Windows Authentication Flow:
User logs into Windows (AD authenticates)
         │
         ▼
Application connects: "Server=SQL01;Integrated Security=True"
         │
         ▼
SQL Server: "Windows says this is DOMAIN\JohnDoe — I trust that"
         │
         ▼
SQL Server maps Windows identity to a Login → grants access
```
```sql
-- Create a Windows login (individual user)
CREATE LOGIN [DOMAIN\JohnDoe] FROM WINDOWS
WITH DEFAULT_DATABASE = OrdersDB;

-- Create a Windows login from an AD group (preferred — manage in AD)
CREATE LOGIN [DOMAIN\SQL_Admins] FROM WINDOWS;
CREATE LOGIN [DOMAIN\AppService] FROM WINDOWS;

-- View Windows logins
SELECT name, type_desc, is_disabled, create_date
FROM sys.server_principals
WHERE type IN ('U', 'G')  -- U = Windows user, G = Windows group
ORDER BY name;
```
**Advantages of Windows Authentication:**

- Password managed by Active Directory (complexity, expiry, lockout)
- Single Sign-On (SSO) — no separate SQL password to manage
- Kerberos-based — more secure than password over network
- Audit trail in Active Directory
- **Microsoft's recommended authentication mode**
### SQL Server Authentication
SQL Server manages its own logins and passwords, stored in the master database. The application passes a username and password directly to SQL Server.

```
SQL Server Authentication Flow:
Application connects: "Server=SQL01;User Id=AppUser;Password=P@ssw0rd"
         │
         ▼
SQL Server validates username + password against its own store
         │
         ▼
Grants or denies access based on login configuration
```
```sql
-- Create a SQL Server login
CREATE LOGIN AppUser
WITH PASSWORD = 'Str0ng!P@ssw0rd#2024',
     MUST_CHANGE = OFF,           -- don't force password change on first login
     CHECK_EXPIRATION = ON,       -- enforce password expiration policy
     CHECK_POLICY = ON,           -- enforce Windows password complexity policy
     DEFAULT_DATABASE = OrdersDB;

-- Modify a login
ALTER LOGIN AppUser WITH PASSWORD = 'NewP@ssw0rd#2024';
ALTER LOGIN AppUser DISABLE;      -- lock out without deleting
ALTER LOGIN AppUser ENABLE;

-- Unlock after too many failed attempts
ALTER LOGIN AppUser WITH PASSWORD = 'NewP@ssw0rd' UNLOCK;

-- View SQL logins
SELECT
    name,
    type_desc,
    is_disabled,
    is_expiration_checked,
    is_policy_checked,
    create_date,
    modify_date
FROM sys.sql_logins
ORDER BY name;
```
**Special SQL Server logins:**

```sql
-- 'sa' (System Administrator) — built-in superuser
-- Best practice: rename and disable sa in production
ALTER LOGIN sa WITH NAME = [sa_disabled];
ALTER LOGIN [sa_disabled] DISABLE;

-- Check if sa is enabled (security audit)
SELECT name, is_disabled
FROM sys.sql_logins
WHERE name IN ('sa', 'sa_disabled');
```
### Authentication Mode Setting
```sql
-- Check current authentication mode
SELECT SERVERPROPERTY('IsIntegratedSecurityOnly') AS windows_only;
-- 1 = Windows Authentication only
-- 0 = Mixed Mode (Windows + SQL Server)

-- Change authentication mode (requires SQL Server restart)
-- Done via SSMS: right-click server → Properties → Security
-- Or via registry (requires restart):
EXEC xp_instance_regwrite
    N'HKEY_LOCAL_MACHINE',
    N'Software\Microsoft\MSSQLServer\MSSQLServer',
    N'LoginMode',
    REG_DWORD,
    2;  -- 1 = Windows only, 2 = Mixed mode
```
### Windows vs SQL Server Authentication Comparison
| Feature | Windows Auth | SQL Server Auth |
| ----- | ----- | ----- |
| Password management | Active Directory | SQL Server |
| Single Sign-On | ✅ Yes | ❌ No |
| Password complexity | AD enforced | SQL policy enforced |
| Network transport | Kerberos / NTLM | Username + password |
| Works without AD | ❌ No | ✅ Yes |
| Recommended by Microsoft | ✅ Yes | Only when needed |
| Best for | Domain-joined environments | External apps, non-domain |
| Audit trail | AD + SQL Server | SQL Server only |
---

## 2. Roles (Server Roles vs Database Roles)
Roles are **named collections of permissions** that simplify permission management. Instead of granting permissions to individuals, you assign them to roles and put users in roles.

### Server Roles
Server roles grant **instance-level permissions** — they control what a login can do across the entire SQL Server instance.

#### Fixed Server Roles
Built-in roles that cannot be modified:

```sql
-- View all fixed server roles
SELECT name, description FROM sys.server_principals
WHERE type = 'R' AND principal_id < 100;
```
| Role | Permissions |
| ----- | ----- |
|  | Unrestricted — can do everything on the instance |
|  | Configure server settings, run SHUTDOWN |
|  | Manage logins and GRANT/REVOKE server permissions |
|  | Manage SQL Server processes (KILL) |
|  | Manage linked servers |
|  | Run BULK INSERT |
|  | Manage disk files |
|  | Create, alter, drop, restore databases |
|  | Default role — all logins are members |
```sql
-- Add login to fixed server role
ALTER SERVER ROLE sysadmin ADD MEMBER [DOMAIN\SQL_Admins];
ALTER SERVER ROLE dbcreator ADD MEMBER DevUser;

-- Remove from server role
ALTER SERVER ROLE dbcreator DROP MEMBER DevUser;

-- Check server role membership
SELECT
    sr.name         AS role_name,
    sp.name         AS member_name,
    sp.type_desc
FROM sys.server_role_members srm
JOIN sys.server_principals sr ON srm.role_principal_id = sr.principal_id
JOIN sys.server_principals sp ON srm.member_principal_id = sp.principal_id
ORDER BY sr.name, sp.name;
```
#### User-Defined Server Roles (SQL Server 2012+)
```sql
-- Create custom server role
CREATE SERVER ROLE MonitoringRole;

-- Grant server-level permissions to the role
GRANT VIEW SERVER STATE TO MonitoringRole;   -- see DMVs
GRANT VIEW ANY DATABASE TO MonitoringRole;   -- see database metadata

-- Add members
ALTER SERVER ROLE MonitoringRole ADD MEMBER MonitoringUser;
```
### Database Roles
Database roles grant **database-level permissions** — they control what a user can do within a specific database.

#### Fixed Database Roles
```sql
-- View fixed database roles
SELECT name FROM sys.database_principals
WHERE type = 'R' AND is_fixed_role = 1;
```
| Role | Permissions |
| ----- | ----- |
|  | Full control of the database |
|  | Manage database roles and permissions |
|  | Add/remove database users |
|  | Back up the database |
|  | Run DDL (CREATE, ALTER, DROP) — no data access |
|  | INSERT, UPDATE, DELETE on all user tables |
|  | SELECT on all user tables |
|  | Explicitly denied write to all user tables |
|  | Explicitly denied read from all user tables |
|  | Default — all users are members |
```sql
-- Create database user from login
USE OrdersDB;
CREATE USER AppUser FOR LOGIN AppUser;        -- same name as login
CREATE USER ReportUser FOR LOGIN ReportUser
    WITH DEFAULT_SCHEMA = Reporting;          -- custom default schema

-- Add to database role
ALTER ROLE db_datareader ADD MEMBER ReportUser;
ALTER ROLE db_datawriter ADD MEMBER AppUser;

-- Remove from role
ALTER ROLE db_datareader DROP MEMBER ReportUser;

-- Check database role membership
SELECT
    dp.name    AS role_name,
    mp.name    AS member_name,
    mp.type_desc
FROM sys.database_role_members drm
JOIN sys.database_principals dp ON drm.role_principal_id = dp.principal_id
JOIN sys.database_principals mp ON drm.member_principal_id = mp.principal_id
ORDER BY dp.name, mp.name;
```
#### User-Defined Database Roles
```sql
-- Create custom database role
CREATE ROLE OrdersReadWrite;

-- Grant permissions to the role
GRANT SELECT, INSERT, UPDATE ON dbo.Orders TO OrdersReadWrite;
GRANT SELECT ON dbo.Customers TO OrdersReadWrite;
GRANT EXECUTE ON dbo.usp_CreateOrder TO OrdersReadWrite;

-- Add members to the role
ALTER ROLE OrdersReadWrite ADD MEMBER AppUser;
ALTER ROLE OrdersReadWrite ADD MEMBER ServiceAccount;

-- Drop role (remove all members first)
ALTER ROLE OrdersReadWrite DROP MEMBER AppUser;
DROP ROLE OrdersReadWrite;
```
### Permission Hierarchy
```
Instance Level (Server Principals — Logins)
    │
    ▼
Database Level (Database Principals — Users)
    │
    ▼
Schema Level (Schema permissions)
    │
    ▼
Object Level (Table, View, Procedure permissions)
    │
    ▼
Row Level (Row Level Security policies)
    │
    ▼
Column Level (Column permissions / Dynamic Data Masking)
```
---

## 3. Row Level Security (RLS)
Row Level Security automatically filters or blocks rows based on the **identity or role of the executing user**. The filter is applied as a hidden predicate on every query — completely transparent to the application.

### How RLS Works
```
Without RLS:
SELECT * FROM Orders → Returns ALL 1,000,000 rows

With RLS (user = 'SalesRep_John'):
SELECT * FROM Orders → SQL Server adds hidden WHERE clause
→ SELECT * FROM Orders WHERE SalesRepID = USER_ID()
→ Returns only John's 500 orders
```
### Filter Predicates vs Block Predicates
**Filter Predicate** — silently filters rows on SELECT, UPDATE, DELETE. Rows that don't match the predicate are **invisible** to the user.

**Block Predicate** — prevents INSERT/UPDATE/DELETE of rows that would violate the predicate. More restrictive.

```sql
-- Step 1: Create the security predicate function
-- Must be SCHEMABINDING inline TVF
CREATE FUNCTION Security.fn_OrdersFilter
(
    @SalesRepID INT
)
RETURNS TABLE
WITH SCHEMABINDING
AS
RETURN
(
    SELECT 1 AS fn_securitypredicate_result
    WHERE
        -- Managers see all rows
        IS_MEMBER('SalesManagers') = 1
        OR
        -- Sales reps see only their own orders
        @SalesRepID = CAST(SESSION_CONTEXT(N'SalesRepID') AS INT)
        OR
        -- Service accounts see all rows
        USER_NAME() IN ('ETLService', 'ReportingService')
);

-- Step 2: Create the security policy
CREATE SECURITY POLICY OrdersSecurityPolicy
ADD FILTER PREDICATE Security.fn_OrdersFilter(SalesRepID)
    ON dbo.Orders,
ADD BLOCK PREDICATE Security.fn_OrdersFilter(SalesRepID)
    ON dbo.Orders AFTER INSERT   -- prevent inserting rows for other reps
WITH (STATE = ON);               -- policy is active immediately
```
### Setting Session Context (for application-level identity)
```sql
-- Application sets user context before querying
-- (e.g., in a multi-tenant web app where all use same DB user)
EXEC sp_set_session_context
    @key   = N'SalesRepID',
    @value = 42,
    @read_only = 1;  -- prevent the session from changing it

-- Now queries are filtered for SalesRepID = 42
SELECT * FROM Orders;
-- Returns only orders where SalesRepID = 42
```
### Multi-Tenant RLS Pattern
```sql
-- Multi-tenant: each tenant sees only their own data
CREATE FUNCTION Security.fn_TenantFilter(@TenantID INT)
RETURNS TABLE WITH SCHEMABINDING AS
RETURN
(
    SELECT 1 AS result
    WHERE @TenantID = CAST(SESSION_CONTEXT(N'TenantID') AS INT)
);

-- Apply to ALL tenant-isolated tables
CREATE SECURITY POLICY TenantIsolationPolicy
ADD FILTER PREDICATE Security.fn_TenantFilter(TenantID) ON dbo.Orders,
ADD FILTER PREDICATE Security.fn_TenantFilter(TenantID) ON dbo.Customers,
ADD FILTER PREDICATE Security.fn_TenantFilter(TenantID) ON dbo.Invoices,
ADD FILTER PREDICATE Security.fn_TenantFilter(TenantID) ON dbo.Products
WITH (STATE = ON);
```
### Managing RLS Policies
```sql
-- Disable policy temporarily (e.g., for maintenance)
ALTER SECURITY POLICY OrdersSecurityPolicy WITH (STATE = OFF);

-- Re-enable
ALTER SECURITY POLICY OrdersSecurityPolicy WITH (STATE = ON);

-- View all security policies
SELECT
    sp.name              AS policy_name,
    sp.is_enabled,
    sp.type_desc,
    OBJECT_NAME(pred.target_object_id) AS table_name,
    pred.predicate_type_desc,
    pred.predicate_definition
FROM sys.security_policies sp
JOIN sys.security_predicates pred ON sp.object_id = pred.object_id;

-- Drop policy
DROP SECURITY POLICY OrdersSecurityPolicy;
DROP FUNCTION Security.fn_OrdersFilter;
```
### RLS Performance Considerations
```sql
-- ✅ Index the columns used in RLS predicates
CREATE INDEX IX_Orders_SalesRepID ON Orders (SalesRepID);
CREATE INDEX IX_Orders_TenantID   ON Orders (TenantID);

-- ✅ Keep predicate functions simple — they run for every row
-- ❌ Avoid subqueries or joins in predicates — executed per-row

-- Check RLS is working (verify rows are filtered)
-- Connect as SalesRep, then:
SELECT COUNT(*) FROM Orders;            -- should show only rep's rows
SELECT COUNT(*) FROM Orders WITH (NOLOCK); -- RLS applies even with NOLOCK
-- RLS bypassed ONLY by sysadmin or db_owner or WITH (NOLOCK) -- NO!
-- Actually RLS is NOT bypassed by NOLOCK — it always applies
-- ONLY sysadmin / db_owner can bypass (they're exempt by default)

-- Make RLS apply even to db_owner (use with extreme care)
ALTER SECURITY POLICY OrdersSecurityPolicy
WITH (STATE = ON, SCHEMABINDING = ON);
```
---

## 4. Dynamic Data Masking
Dynamic Data Masking (DDM) **obfuscates sensitive column values** for unauthorized users at query time. The data is stored unmasked — only the presentation is masked. Privileged users always see the real data.

```
Without DDM (privileged user):
SELECT * FROM Customers
→ CustomerID | Name      | Email              | CreditCard
  1          | John Doe  | john@example.com   | 4111-1111-1111-1234

With DDM (unprivileged user):
SELECT * FROM Customers
→ CustomerID | Name      | Email              | CreditCard
  1          | XXXX      | XXXX@XXXX.com      | XXXX-XXXX-XXXX-1234
```
### Mask Types
```sql
-- 1. DEFAULT mask — full masking based on data type
-- Strings → XXXX, Numbers → 0, Dates → 1900-01-01
ALTER TABLE Customers
ADD MASKED WITH (FUNCTION = 'default()') ON CustomerSSN;

-- 2. PARTIAL mask — show first/last N characters
-- partial(prefix, padding, suffix)
ALTER TABLE Customers
ADD MASKED WITH (FUNCTION = 'partial(0,"XXXX-XXXX-XXXX-",4)') ON CreditCard;
-- Shows: XXXX-XXXX-XXXX-1234

ALTER TABLE Customers
ADD MASKED WITH (FUNCTION = 'partial(1,"XXXXXXXXXXXX",1)') ON Name;
-- Shows: J XXXXXXXXXXXX e (first and last char of name)

-- 3. EMAIL mask — shows first letter + XXX@domain.com
ALTER TABLE Customers
ADD MASKED WITH (FUNCTION = 'email()') ON Email;
-- john@example.com → jXXX@XXXX.com

-- 4. RANDOM mask — for numeric columns (random number in range)
ALTER TABLE Patients
ADD MASKED WITH (FUNCTION = 'random(1, 12)') ON BirthMonth;
-- Returns random number between 1 and 12
```
### Adding and Removing Masks
```sql
-- Create table with masks defined inline
CREATE TABLE Customers (
    CustomerID  INT            NOT NULL PRIMARY KEY,
    Name        NVARCHAR(100)  MASKED WITH (FUNCTION = 'partial(1,"XXXXXX",1)'),
    Email       VARCHAR(200)   MASKED WITH (FUNCTION = 'email()'),
    Phone       VARCHAR(20)    MASKED WITH (FUNCTION = 'default()'),
    SSN         CHAR(11)       MASKED WITH (FUNCTION = 'partial(0,"XXX-XX-",4)'),
    CreditCard  CHAR(19)       MASKED WITH (FUNCTION = 'partial(0,"XXXX-XXXX-XXXX-",4)'),
    Salary      DECIMAL(18,2)  MASKED WITH (FUNCTION = 'default()')
);

-- Add mask to existing column
ALTER TABLE Customers
ALTER COLUMN Phone ADD MASKED WITH (FUNCTION = 'default()');

-- Remove mask from a column
ALTER TABLE Customers
ALTER COLUMN Phone DROP MASKED;

-- Change mask function
ALTER TABLE Customers
ALTER COLUMN CreditCard
ADD MASKED WITH (FUNCTION = 'partial(0,"XXXX-XXXX-XXXX-",4)');
```
### Granting Unmask Permission
```sql
-- Grant permission to see unmasked data
GRANT UNMASK TO PowerUser;
GRANT UNMASK ON dbo.Customers TO SeniorAnalyst;  -- table-level (SQL 2022+)
GRANT UNMASK ON dbo.Customers(SSN) TO AuditUser; -- column-level (SQL 2022+)

-- Revoke unmask
REVOKE UNMASK FROM PowerUser;

-- View masked columns
SELECT
    t.name   AS table_name,
    c.name   AS column_name,
    c.masking_function
FROM sys.masked_columns c
JOIN sys.tables t ON c.object_id = t.object_id;
```
### DDM Limitations
```sql
-- ⚠️ DDM does NOT prevent inference attacks
-- A user can guess masked values:
SELECT * FROM Customers WHERE SSN = '123-45-6789';
-- If this returns a row, they've confirmed the SSN — even though display is masked

-- ⚠️ DDM does NOT encrypt data — storage is still plaintext
-- Anyone with direct file access or sysadmin sees real data

-- ✅ DDM is best for: limiting casual exposure in application layer
-- Combine with RLS, encryption, and permissions for full security
```
---

## 5. Transparent Data Encryption (TDE)
TDE encrypts the **physical data files** (.mdf, .ndf, .ldf, and backups) at rest. If someone steals the hard drives or backup files, they cannot read the data without the encryption key. The encryption/decryption is completely transparent to applications — no code changes required.

```
Without TDE:
Attacker steals .mdf file → attaches to their SQL Server → reads all data ❌

With TDE:
Attacker steals .mdf file → attaches to their SQL Server → encrypted gibberish ✅
                             (without the certificate + private key)
```
### TDE Architecture
```
TDE Encryption Hierarchy
──────────────────────────────────────────────────────
Service Master Key (SMK)
    └── protected by Windows DPAPI + machine key
         │
         ▼
Database Master Key (DMK) in master database
    └── protected by SMK
         │
         ▼
Certificate (in master database)
    └── protected by DMK
         │
         ▼
Database Encryption Key (DEK) in each user database
    └── protected by the Certificate
         │
         ▼
Encrypted data pages (.mdf / .ndf / .ldf / backups)
```
### Enabling TDE Step by Step
```sql
-- Step 1: Create Database Master Key in master database
USE master;
CREATE MASTER KEY ENCRYPTION BY PASSWORD = 'Str0ng!MasterKey#2024';

-- Step 2: Create certificate to protect the DEK
CREATE CERTIFICATE TDE_Certificate
WITH SUBJECT = 'TDE Certificate for OrdersDB',
     EXPIRY_DATE = '2030-01-01';

-- Step 3: CRITICAL — back up the certificate immediately!
-- Without this backup, encrypted databases cannot be restored elsewhere
BACKUP CERTIFICATE TDE_Certificate
TO FILE = 'C:\SecureBackup\TDE_Certificate.cer'
WITH PRIVATE KEY (
    FILE = 'C:\SecureBackup\TDE_Certificate.pvk',
    ENCRYPTION BY PASSWORD = 'CertBackupP@ssw0rd!'
);

-- Step 4: Create DEK in the user database
USE OrdersDB;
CREATE DATABASE ENCRYPTION KEY
WITH ALGORITHM = AES_256             -- AES_128, AES_192, AES_256, TRIPLE_DES_3KEY
ENCRYPTION BY SERVER CERTIFICATE TDE_Certificate;

-- Step 5: Enable encryption
ALTER DATABASE OrdersDB SET ENCRYPTION ON;
-- Background encryption process starts — encrypts all existing pages
-- Database is fully functional during encryption

-- Step 6: Verify encryption status
SELECT
    db.name,
    dek.encryption_state_desc,
    dek.percent_complete,           -- progress during initial encryption
    dek.encryptor_type,
    dek.key_algorithm,
    dek.key_length
FROM sys.dm_database_encryption_keys dek
JOIN sys.databases db ON dek.database_id = db.database_id;
-- encryption_state: 0=None, 1=Unencrypted, 2=Encrypting,
--                   3=Encrypted, 4=Key change, 5=Decrypting
```
### TDE and Backups
```sql
-- TDE-encrypted databases produce encrypted backups automatically
BACKUP DATABASE OrdersDB TO DISK = 'D:\Backups\OrdersDB.bak';
-- Backup is encrypted — cannot restore without the certificate

-- Restore on another server: must install certificate FIRST
USE master;
CREATE MASTER KEY ENCRYPTION BY PASSWORD = 'RestoredMasterKey#2024';

CREATE CERTIFICATE TDE_Certificate
FROM FILE = 'C:\SecureBackup\TDE_Certificate.cer'
WITH PRIVATE KEY (
    FILE = 'C:\SecureBackup\TDE_Certificate.pvk',
    DECRYPTION BY PASSWORD = 'CertBackupP@ssw0rd!'
);

-- Now restore succeeds
RESTORE DATABASE OrdersDB FROM DISK = 'D:\Backups\OrdersDB.bak';
```
### Disabling TDE
```sql
-- Remove encryption (background decryption process runs)
ALTER DATABASE OrdersDB SET ENCRYPTION OFF;

-- Wait for decryption to complete
-- Then remove the DEK
DROP DATABASE ENCRYPTION KEY;

-- Verify fully decrypted
SELECT encryption_state_desc FROM sys.dm_database_encryption_keys
WHERE database_id = DB_ID('OrdersDB');
-- Should show 'NOT_ENCRYPTED' or be absent
```
---

## 6. Column-Level Encryption
While TDE encrypts entire files, **column-level encryption** (also called cell-level encryption) encrypts specific columns in a table. The data is stored encrypted — even within SQL Server, only authorized code with the right key can decrypt it. Useful for protecting specific sensitive fields like SSN, credit card numbers, or health data.

### Symmetric Key Encryption
```sql
-- Step 1: Create master key (if not already exists)
USE OrdersDB;
CREATE MASTER KEY ENCRYPTION BY PASSWORD = 'Str0ng!DBMasterKey';

-- Step 2: Create certificate to protect the symmetric key
CREATE CERTIFICATE ColumnEncryptionCert
WITH SUBJECT = 'Column Encryption Certificate';

-- Step 3: Create symmetric key (used for actual encryption)
CREATE SYMMETRIC KEY ColumnEncryptionKey
WITH ALGORITHM = AES_256
ENCRYPTION BY CERTIFICATE ColumnEncryptionCert;

-- Step 4: Add encrypted column to table
ALTER TABLE Customers ADD SSN_Encrypted VARBINARY(256);

-- Step 5: Populate encrypted column
-- Must open key, encrypt, then close key
OPEN SYMMETRIC KEY ColumnEncryptionKey
DECRYPTION BY CERTIFICATE ColumnEncryptionCert;

UPDATE Customers
SET SSN_Encrypted = ENCRYPTBYKEY(
    KEY_GUID('ColumnEncryptionKey'),
    SSN,                              -- plaintext column
    1,                                -- add authenticator (prevents swapping)
    CONVERT(VARBINARY, CustomerID)    -- authenticator value
);

CLOSE SYMMETRIC KEY ColumnEncryptionKey;

-- Step 6: Optionally remove plaintext column
ALTER TABLE Customers DROP COLUMN SSN;
```
### Querying Encrypted Columns
```sql
-- Must open key before querying
OPEN SYMMETRIC KEY ColumnEncryptionKey
DECRYPTION BY CERTIFICATE ColumnEncryptionCert;

SELECT
    CustomerID,
    Name,
    CONVERT(VARCHAR(11),
        DECRYPTBYKEY(
            SSN_Encrypted,
            1,
            CONVERT(VARBINARY, CustomerID)  -- same authenticator used to encrypt
        )
    ) AS SSN_Decrypted
FROM Customers
WHERE CustomerID = 101;

CLOSE SYMMETRIC KEY ColumnEncryptionKey;

-- Searching encrypted data — exact match only
-- (cannot do LIKE or range comparisons on encrypted data)
OPEN SYMMETRIC KEY ColumnEncryptionKey
DECRYPTION BY CERTIFICATE ColumnEncryptionCert;

SELECT * FROM Customers
WHERE CONVERT(VARCHAR(11), DECRYPTBYKEY(SSN_Encrypted, 1,
      CONVERT(VARBINARY, CustomerID))) = '123-45-6789';

CLOSE SYMMETRIC KEY ColumnEncryptionKey;
```
### Always Encrypted (SQL Server 2016+)
Always Encrypted is the **modern approach** to column encryption — keys are stored in the **client application**, not in SQL Server. Even SQL Server administrators cannot see the plaintext data.

```
Traditional Column Encryption:     Always Encrypted:
────────────────────────────────   ─────────────────────────────
Key in SQL Server                  Key in Client / Azure Key Vault
DBAs can open key → see data       DBAs see only ciphertext
Encryption in server memory        Encryption in client driver
```
```sql
-- Always Encrypted columns defined in table
CREATE TABLE Patients (
    PatientID INT PRIMARY KEY,
    Name      NVARCHAR(100),
    SSN       CHAR(11) ENCRYPTED WITH (
        COLUMN_ENCRYPTION_KEY = SSN_CEK,
        ENCRYPTION_TYPE = Deterministic,  -- allows equality search
        ALGORITHM = 'AEAD_AES_256_CBC_HMAC_SHA_256'
    ),
    Diagnosis NVARCHAR(500) ENCRYPTED WITH (
        COLUMN_ENCRYPTION_KEY = Diagnosis_CEK,
        ENCRYPTION_TYPE = Randomized,     -- more secure, no search
        ALGORITHM = 'AEAD_AES_256_CBC_HMAC_SHA_256'
    )
);
```
| Encryption Type | Supports Equality Search | Security Level |
| ----- | ----- | ----- |
|  | ✅ Yes — same plaintext = same ciphertext | Good |
|  | ❌ No — same plaintext = different ciphertext | Better |
---

## 7. GRANT, DENY, REVOKE
These three statements control the **permission system** at every level — server, database, schema, and object.

### GRANT
Gives a principal permission to perform an action.

```sql
-- Server-level GRANTs
GRANT VIEW SERVER STATE TO MonitoringUser;    -- see DMVs
GRANT ALTER ANY LOGIN TO SecurityAdmin;       -- manage logins
GRANT CREATE ANY DATABASE TO DevLead;         -- create databases

-- Database-level GRANTs
GRANT CREATE TABLE TO AppDeveloper;
GRANT CREATE PROCEDURE TO AppDeveloper;
GRANT BACKUP DATABASE TO BackupOperator;

-- Schema-level GRANTs (covers all objects in schema)
GRANT SELECT ON SCHEMA::Sales TO ReportUser;
GRANT SELECT, INSERT, UPDATE ON SCHEMA::dbo TO AppUser;
GRANT EXECUTE ON SCHEMA::dbo TO ServiceAccount;

-- Object-level GRANTs
GRANT SELECT ON dbo.Orders TO ReportUser;
GRANT SELECT ON dbo.Orders(OrderID, OrderDate, TotalAmount)
    TO LimitedUser;                               -- column-level grant
GRANT INSERT, UPDATE ON dbo.Orders TO AppUser;
GRANT EXECUTE ON dbo.usp_CreateOrder TO AppUser;
GRANT VIEW DEFINITION ON dbo.usp_CreateOrder TO Developer;

-- WITH GRANT OPTION — allows the grantee to grant to others
GRANT SELECT ON dbo.Products TO PowerUser WITH GRANT OPTION;
```
### DENY
**Explicitly forbids** a permission. DENY overrides GRANT — even if a user has been granted permission through a role, an explicit DENY blocks it.

```sql
-- DENY overrides any GRANT from any role membership
DENY SELECT ON dbo.SalaryData TO ContractEmployee;
-- Even if ContractEmployee is in db_datareader role,
-- they cannot SELECT from SalaryData

DENY DELETE ON dbo.Orders TO AppUser;         -- can INSERT/UPDATE but not DELETE
DENY ALTER ON SCHEMA::HR TO AppUser;          -- prevent schema modifications
DENY VIEW DEFINITION ON dbo.usp_PayrollCalc TO Auditor;

-- DENY at column level
DENY SELECT ON dbo.Employees(Salary) TO Manager;
-- Manager can SELECT all columns EXCEPT Salary
```
### REVOKE
**Removes** a previously granted or denied permission. REVOKE returns the principal to a neutral state (neither granted nor denied).

```sql
-- Remove a GRANT
REVOKE SELECT ON dbo.Orders FROM ReportUser;

-- Remove a DENY
REVOKE DENY SELECT ON dbo.SalaryData FROM ContractEmployee;
-- Note: user may now have access through role membership

-- Cascade: revoke permissions that were granted WITH GRANT OPTION
REVOKE SELECT ON dbo.Products FROM PowerUser CASCADE;
-- Also removes permissions PowerUser granted to others

-- REVOKE vs DENY:
-- REVOKE: neutral (role membership may still grant access)
-- DENY:   explicit block (overrides all grants)
```
### Permission Precedence
```
DENY (explicit) > GRANT (explicit) > Role membership grant > public grant

Example:
User is member of db_datareader (GRANT SELECT on all tables)
DENY SELECT on Salaries explicitly applied to user
→ User CANNOT select from Salaries (DENY wins)

Remove the DENY (REVOKE):
→ User CAN now select from Salaries (db_datareader grant applies)
```
### Viewing Effective Permissions
```sql
-- Check permissions granted on an object
SELECT
    dp.name             AS principal_name,
    dp.type_desc        AS principal_type,
    perm.state_desc     AS permission_state,
    perm.permission_name,
    OBJECT_NAME(perm.major_id) AS object_name
FROM sys.database_permissions perm
JOIN sys.database_principals dp ON perm.grantee_principal_id = dp.principal_id
WHERE major_id = OBJECT_ID('dbo.Orders')
ORDER BY dp.name, perm.permission_name;

-- Check all permissions for a specific user
SELECT
    perm.state_desc,
    perm.permission_name,
    OBJECT_NAME(perm.major_id) AS object_name,
    perm.class_desc
FROM sys.database_permissions perm
WHERE grantee_principal_id = USER_ID('AppUser');

-- Effective permissions for current user
SELECT * FROM fn_my_permissions(NULL, 'DATABASE');
SELECT * FROM fn_my_permissions('dbo.Orders', 'OBJECT');

-- Check if a login is sysadmin
SELECT IS_SRVROLEMEMBER('sysadmin');      -- 1 = yes, 0 = no, NULL = doesn't exist
SELECT IS_MEMBER('db_datareader');         -- database role check
```
### Ownership Chaining
```sql
-- When a stored procedure and table share the same owner,
-- the user only needs EXECUTE on the procedure — not SELECT on the table
-- This is "ownership chaining"

-- AppUser has EXECUTE on usp_GetOrders but NO SELECT on Orders
-- usp_GetOrders SELECTs from Orders (same owner: dbo)
-- → Works! Ownership chain allows access through the procedure

GRANT EXECUTE ON dbo.usp_GetOrders TO AppUser;
-- AppUser can execute and get results — never granted direct table access
-- ✅ Security best practice: grant EXECUTE on procedures, not direct table access
```
---

## 8. Certificate-Based Security
Certificates in SQL Server are used for **key management, signing, and authentication**. They underpin TDE, column encryption, cross-database permissions, and Service Broker security.

### Creating and Managing Certificates
```sql
-- Create certificate manually
CREATE CERTIFICATE AppCertificate
WITH SUBJECT = 'Application Security Certificate',
     START_DATE = '2024-01-01',
     EXPIRY_DATE = '2027-01-01';

-- Create from file (external CA certificate)
CREATE CERTIFICATE ExternalCert
FROM FILE = 'C:\Certs\external_cert.cer';

-- Create from file with private key
CREATE CERTIFICATE AppCertificate
FROM FILE = 'C:\Certs\app_cert.cer'
WITH PRIVATE KEY (
    FILE = 'C:\Certs\app_cert_key.pvk',
    DECRYPTION BY PASSWORD = 'KeyP@ssword!'
);

-- Back up a certificate (critical for TDE and encryption)
BACKUP CERTIFICATE AppCertificate
TO FILE = 'C:\Backup\AppCertificate.cer'
WITH PRIVATE KEY (
    FILE = 'C:\Backup\AppCertificate_key.pvk',
    ENCRYPTION BY PASSWORD   = 'BackupKeyP@ss!',
    DECRYPTION BY PASSWORD   = 'KeyP@ssword!'
);

-- View certificates
SELECT
    name,
    subject,
    start_date,
    expiry_date,
    pvt_key_encryption_type_desc,
    thumbprint
FROM sys.certificates
ORDER BY expiry_date;
```
### Code Signing with Certificates
Certificates can **sign stored procedures and modules**, granting elevated permissions to code without granting them to users. This is a powerful security pattern for privilege escalation control.

```sql
-- Problem: AppUser needs to access HR.Salaries but shouldn't have direct access
-- Solution: Sign the procedure with a certificate that has the permission

-- Step 1: Create certificate
CREATE CERTIFICATE HRAccessCert
WITH SUBJECT = 'HR Access for Payroll Procedure';

-- Step 2: Create user from certificate (no login — security principal only)
CREATE USER HRAccessUser FROM CERTIFICATE HRAccessCert;

-- Step 3: Grant permission to the certificate user
GRANT SELECT ON HR.Salaries TO HRAccessUser;

-- Step 4: Sign the stored procedure with the certificate
ADD SIGNATURE TO dbo.usp_GetPayroll
BY CERTIFICATE HRAccessCert;

-- Step 5: Grant EXECUTE on procedure to AppUser (no direct table access)
GRANT EXECUTE ON dbo.usp_GetPayroll TO AppUser;

-- Result: AppUser can EXECUTE usp_GetPayroll and get salary data
-- But AppUser cannot SELECT directly from HR.Salaries
-- The certificate elevates permissions only within the signed procedure
```
### Certificate-Based Login (SQL Server 2022+)
```sql
-- Map a certificate to a server login (for service authentication)
CREATE LOGIN CertLogin FROM CERTIFICATE AppCertificate;
GRANT VIEW SERVER STATE TO CertLogin;
```
### Key Management Hierarchy
```
Service Master Key (SMK)
├── Created automatically by SQL Server
├── Protected by Windows DPAPI (machine-bound)
├── Used to protect Database Master Keys
└── Back up: BACKUP SERVICE MASTER KEY TO FILE = '...'

Database Master Key (DMK)
├── Created manually per database
├── Protected by SMK (auto-open) or password
├── Used to protect certificates and asymmetric keys
└── Back up: BACKUP MASTER KEY TO FILE = '...'

Certificate
├── Protected by DMK
├── Used to protect symmetric keys (column encryption)
├── Used for TDE (protects DEK)
├── Used for code signing
└── Has public/private key pair

Symmetric Key
├── Protected by certificate (or password/asymmetric key)
├── Used for actual data encryption (fast)
└── AES_256 recommended
```
```sql
-- Back up Service Master Key (do once after SQL Server install)
BACKUP SERVICE MASTER KEY
TO FILE = 'C:\SecureBackup\SMK_Backup.key'
ENCRYPTION BY PASSWORD = 'SMK_BackupP@ss!';

-- Back up Database Master Key
USE OrdersDB;
BACKUP MASTER KEY
TO FILE = 'C:\SecureBackup\OrdersDB_DMK.key'
ENCRYPTION BY PASSWORD = 'DMK_BackupP@ss!';

-- Restore Service Master Key (disaster recovery)
RESTORE SERVICE MASTER KEY
FROM FILE = 'C:\SecureBackup\SMK_Backup.key'
DECRYPTION BY PASSWORD = 'SMK_BackupP@ss!';
```
### Certificate Expiry Monitoring
```sql
-- Alert on expiring certificates
SELECT
    name                             AS cert_name,
    subject,
    expiry_date,
    DATEDIFF(DAY, GETDATE(), expiry_date) AS days_until_expiry,
    CASE
        WHEN expiry_date < GETDATE()          THEN '🔴 EXPIRED'
        WHEN expiry_date < DATEADD(DAY, 30, GETDATE()) THEN '🟡 EXPIRING SOON'
        ELSE '🟢 OK'
    END                              AS status
FROM sys.certificates
ORDER BY expiry_date;
```
---

## Security Audit Queries
```sql
-- 1. Find all sysadmin members
SELECT sp.name AS login_name, sp.type_desc, sp.is_disabled
FROM sys.server_role_members srm
JOIN sys.server_principals sp ON srm.member_principal_id = sp.principal_id
WHERE srm.role_principal_id = SYSUSER_ID('sysadmin');

-- 2. Find logins with passwords that never expire
SELECT name, is_expiration_checked, is_policy_checked
FROM sys.sql_logins
WHERE is_expiration_checked = 0 AND is_disabled = 0;

-- 3. Databases without TDE
SELECT name FROM sys.databases
WHERE database_id NOT IN (
    SELECT database_id FROM sys.dm_database_encryption_keys
    WHERE encryption_state = 3  -- 3 = Encrypted
)
AND name NOT IN ('master', 'model', 'msdb', 'tempdb');

-- 4. All RLS policies in the database
SELECT sp.name, sp.is_enabled, OBJECT_NAME(pred.target_object_id)
FROM sys.security_policies sp
JOIN sys.security_predicates pred ON sp.object_id = pred.object_id;

-- 5. Tables with masked columns
SELECT t.name, c.name, c.masking_function
FROM sys.masked_columns c
JOIN sys.tables t ON c.object_id = t.object_id;

-- 6. All permissions granted/denied to a user
SELECT state_desc, permission_name, class_desc,
       OBJECT_NAME(major_id) AS object_name
FROM sys.database_permissions
WHERE grantee_principal_id = USER_ID('AppUser')
ORDER BY class_desc, object_name;
```
---

## Security Quick Reference
| Feature | Purpose | Scope | Key Command |
| ----- | ----- | ----- | ----- |
| **Windows Auth** | Identity via Active Directory | Instance |  |
| **SQL Auth** | Identity via SQL password | Instance |  |
| **Server Roles** | Instance-level permissions | Instance |  |
| **Database Roles** | Database-level permissions | Database |  |
| **GRANT** | Give permission | Any level |  |
| **DENY** | Block permission (overrides GRANT) | Any level |  |
| **REVOKE** | Remove grant or deny | Any level |  |
| **RLS** | Filter rows by user identity | Row level |  |
| **DDM** | Mask column values | Column level |  |
| **TDE** | Encrypt data files at rest | File level |  |
| **Column Encryption** | Encrypt specific columns | Column level |  |
| **Always Encrypted** | Client-side column encryption | Column level | Column  |
| **Certificates** | Key management, code signing | Various |  |
| **Code Signing** | Elevate procedure permissions | Object level |  |
## Security Layering Best Practices
```
Recommended Security Stack for Production OLTP:
────────────────────────────────────────────────────────────────
1. Use Windows Authentication (AD groups → SQL logins)
2. Disable sa login; rename if needed
3. Least privilege: db_datareader/writer for apps, not db_owner
4. Grant EXECUTE on procedures — not direct table access
5. Enable TDE on all production databases
6. Enable RLS for multi-tenant or role-filtered data
7. Apply DDM for sensitive columns exposed to reporting users
8. Audit with SQL Server Audit or Extended Events
9. Back up all certificates and keys — store securely offsite
10. Monitor certificate expiry — rotate before expiry
────────────────────────────────────────────────────────────────
```




<!--- Eraser file: https://app.eraser.io/workspace/63gHFhnHXj5u7JccNU20 --->