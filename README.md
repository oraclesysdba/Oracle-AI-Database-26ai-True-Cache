# Oracle AI Database 26ai — True Cache Setup Guide

> **Release:** 23.26.1+ · **Platform:** Linux x86-64 (On-Premises) · **Updated:** March 2026

True Cache is a **diskless, read-only Oracle instance** that maintains a transactionally consistent replica of the primary database entirely in memory using Redo Apply. Unlike Redis or Memcached, True Cache is fully ACID-compliant — and the JDBC driver routes read-only transactions to it **automatically with zero application code changes**.

---

## Table of Contents

- [How True Cache Works](#how-true-cache-works)
- [True Cache vs Active Data Guard](#true-cache-vs-active-data-guard)
- [Prerequisites](#prerequisites)
- [Phase 1 — Configure the Primary Database](#phase-1--configure-the-primary-database)
- [Phase 2 — Create the True Cache Instance](#phase-2--create-the-true-cache-instance)
- [Phase 3 — Create the True Cache Service](#phase-3--create-the-true-cache-service)
- [Phase 4 — Application Connection](#phase-4--application-connection)
- [Phase 5 — Monitoring & Management](#phase-5--monitoring--management)
- [True Cache vs Standby — Step-by-Step Comparison](#true-cache-vs-standby--step-by-step-comparison)
- [Setup Checklist](#setup-checklist)
- [Quick Reference](#quick-reference)

---

## How True Cache Works

```
┌─────────────────┐        READ (auto-routed)       ┌──────────────────────────┐
│   App Server    │ ──────────────────────────────► │      True Cache          │
│  JDBC / UCP     │                                  │  Diskless Oracle Instance│
│                 │        WRITE (auto-routed)       │  SGA Buffer Cache only   │
│                 │ ──────────────────────────────► │  READ ONLY WITH APPLY    │
└─────────────────┘             │                   └──────────────────────────┘
                                │                              ▲
                                ▼                              │ Redo Stream (async)
                   ┌────────────────────────┐                  │
                   │     Primary DB         │ ─────────────────┘
                   │   Oracle 26ai          │
                   │   READ WRITE           │
                   │   ASM disk storage     │
                   └────────────────────────┘
```

**Key facts:**
- True Cache has **no datafiles** — all data lives in the SGA buffer cache
- Blocks not yet cached are **fetched on-demand** from the primary, then cached for subsequent queries
- Apply lag is typically **milliseconds** — consistent with primary at commit level
- **One DBCA command** (`-createTrueCache`) handles all wiring automatically

---

## True Cache vs Active Data Guard

| Feature | True Cache | Active Data Guard |
|---|---|---|
| Primary purpose | Read offload / low latency | HA failover + read offload |
| Storage on standby | **None (diskless)** | Full ASM disk copy |
| RMAN restore needed | **No** | Yes |
| Standby redo logs | **No** | Yes |
| Failover capable | No | Yes |
| ACID consistent | Yes | Yes |
| JDBC auto-routing | **Yes (zero app change)** | Manual service routing |
| License | Active Data Guard option | Active Data Guard option |
| MRP process | Auto-managed | Manual start required |
| RMAN backups | Not applicable | Supported / offloadable |

---

## Prerequisites

### Software requirements

| Requirement | Detail | Status |
|---|---|---|
| Oracle DB version | Oracle AI Database 26ai (23.26.1+) on primary | **Required** |
| True Cache binary | Same Oracle version as primary | **Required** |
| License | Active Data Guard option (included in EE) | **Required** |
| JDBC driver | Oracle JDBC 26ai thin driver | **Required** for auto-routing |
| Grid Infrastructure | Oracle GI 26ai on True Cache host | Recommended |

### Primary database requirements

| Requirement | Detail |
|---|---|
| ARCHIVELOG mode | Must be enabled |
| FORCE LOGGING | Must be enabled — prevents unlogged direct writes |
| DB_UNIQUE_NAME | Must be set on primary |
| Redo transport | ASYNC or SYNC supported |
| Password file | Required for redo transport authentication |

### True Cache host requirements

| Requirement | Detail |
|---|---|
| **Disk storage** | **None needed** — True Cache is entirely diskless |
| Memory (SGA) | Size to hold your read working set — larger = higher hit rate |
| Network | Low-latency link to primary (10 GbE minimum recommended) |
| OS / kernel | Same OS version as primary (Linux x86-64 for on-prem GA) |
| Oracle Home | Full Oracle DB 26ai software install (same version as primary) |

---

## Phase 1 — Configure the Primary Database

> All steps in this phase run on the **PRIMARY database host** as SYSDBA unless noted.

---

### Step 1 — Verify and enable ARCHIVELOG mode

```sql
-- Check current mode
SELECT LOG_MODE FROM V$DATABASE;

-- If NOARCHIVELOG, enable it (requires restart)
SHUTDOWN IMMEDIATE;
STARTUP MOUNT;
ALTER DATABASE ARCHIVELOG;
ALTER DATABASE OPEN;

-- Verify
SELECT LOG_MODE FROM V$DATABASE;
-- Expected: ARCHIVELOG
```

---

### Step 2 — Enable FORCE LOGGING

```sql
ALTER DATABASE FORCE LOGGING;

-- Verify
SELECT FORCE_LOGGING FROM V$DATABASE;
-- Expected: YES
```

---

### Step 3 — Set DB_UNIQUE_NAME

```sql
ALTER SYSTEM SET DB_UNIQUE_NAME='PRODDB' SCOPE=SPFILE;
-- Requires DB restart to take effect if changed
```

---

### Step 4 — Configure redo transport to True Cache

> Replace `TRUECACHE1` with your chosen `DB_UNIQUE_NAME` for the True Cache instance.
> Replace `truecache-host` with the actual hostname or IP.

```sql
-- Add True Cache as redo transport destination
ALTER SYSTEM SET LOG_ARCHIVE_DEST_2=
  'SERVICE=TRUECACHE1
   ASYNC
   VALID_FOR=(ONLINE_LOGFILES,PRIMARY_ROLE)
   DB_UNIQUE_NAME=TRUECACHE1'
  SCOPE=BOTH;

ALTER SYSTEM SET LOG_ARCHIVE_DEST_STATE_2=ENABLE SCOPE=BOTH;

-- Register both DBs in DG_CONFIG
ALTER SYSTEM SET LOG_ARCHIVE_CONFIG=
  'DG_CONFIG=(PRODDB,TRUECACHE1)' SCOPE=BOTH;
```

---

### Step 5 — Create and distribute the password file

```bash
# On PRIMARY OS (oracle user)
orapwd file=$ORACLE_HOME/dbs/orapwPRODDB \
        password=SysPassword123 \
        entries=10 \
        format=12.2

# Copy password file to True Cache host
scp $ORACLE_HOME/dbs/orapwPRODDB \
    oracle@truecache-host:$ORACLE_HOME/dbs/orapwTRUECACHE1
```

---

### Step 6 — Add True Cache TNS entry on primary

Edit `$ORACLE_HOME/network/admin/tnsnames.ora` on the **PRIMARY** host:

```
TRUECACHE1 =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL=TCP)
               (HOST=truecache-host.example.com)
               (PORT=1521))
    (CONNECT_DATA =
      (SERVICE_NAME=TRUECACHE1)))
```

---

## Phase 2 — Create the True Cache Instance

> All steps in this phase run on the **TRUE CACHE HOST** unless noted.

---

### Step 7 — Install Oracle software on True Cache host

- Install Oracle AI Database 26ai binaries (same version + patches as primary)
- Do **NOT** run `dbca` to create a normal database — you will use `-createTrueCache` in Step 10
- Apply all Oracle patches matching the primary

---

### Step 8 — Create the seed init.ora on True Cache host

Create `$ORACLE_HOME/dbs/initTRUECACHE1.ora`:

```ini
# Seed init.ora — DBCA will expand this
DB_NAME=PRODDB
DB_UNIQUE_NAME=TRUECACHE1

# KEY PARAMETER — makes this a True Cache instance
TRUE_CACHE=TRUE

# Size SGA to hold your read working set
SGA_TARGET=8G
SGA_MAX_SIZE=8G

# Redo source
FAL_SERVER=PRODDB
FAL_CLIENT=TRUECACHE1

LOG_ARCHIVE_CONFIG='DG_CONFIG=(PRODDB,TRUECACHE1)'
LOG_ARCHIVE_DEST_1=
  'SERVICE=PRODDB
   VALID_FOR=(STANDBY_LOGFILES,STANDBY_ROLE)
   DB_UNIQUE_NAME=PRODDB'
```

> **Note:** `TRUE_CACHE=TRUE` is the single parameter that distinguishes this from a normal standby. No `DB_FILE_NAME_CONVERT` needed — there are no datafiles.

---

### Step 9 — Add primary TNS entry on True Cache host

Edit `$ORACLE_HOME/network/admin/tnsnames.ora` on the **TRUE CACHE HOST**:

```
PRODDB =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL=TCP)
               (HOST=primary-host.example.com)
               (PORT=1521))
    (CONNECT_DATA =
      (SERVICE_NAME=PRODDB)))
```

---

### Step 10 — Run DBCA to create the True Cache instance

> This is the key Oracle 26ai command. `-createTrueCache` automates all wiring — no RMAN restore, no standby redo log creation, no manual MRP startup.

```bash
dbca -silent \
  -createTrueCache \
  -gdbName PRODDB \
  -sid TRUECACHE1 \
  -sourceDBConnectionString primary-host:1521/PRODDB \
  -sysDBAUserName sys \
  -sysDBAPassword SysPassword123 \
  -trueCacheSid TRUECACHE1 \
  -trueCacheServiceName tc_service \
  -initParams "sga_target=8g,sga_max_size=8g"
```

**DBCA automatically performs:**
- Connects to primary and retrieves database metadata
- Creates the True Cache Oracle instance (no datafiles created)
- Configures Redo Apply streaming from primary
- Registers `tc_service` on both primary and True Cache
- Starts Redo Apply — buffer cache begins populating

---

### Step 11 — Verify True Cache is running and in sync

```sql
-- On TRUE CACHE instance
SELECT STATUS, DATABASE_ROLE, OPEN_MODE FROM V$DATABASE;
-- Expected: OPEN / PHYSICAL STANDBY / READ ONLY WITH APPLY

-- Check apply lag (should be milliseconds)
SELECT NAME, VALUE, UNIT
FROM   V$DATAGUARD_STATS
WHERE  NAME IN ('apply lag','transport lag');

-- On PRIMARY — confirm True Cache is connected
SELECT DB_UNIQUE_NAME, STATUS, GAP_STATUS
FROM   V$DATAGUARD_STATS
WHERE  DB_UNIQUE_NAME = 'TRUECACHE1';
```

---

## Phase 3 — Create the True Cache Service

> Run on the **PRIMARY database** as SYSDBA.

---

### Step 12 — Create and start the True Cache service

```sql
-- Create service with TRUE_CACHE_SERVICE parameter
EXEC DBMS_SERVICE.CREATE_SERVICE(
  service_name       => 'tc_service',
  network_name       => 'tc_service',
  true_cache_service => 'tc_service'
);

EXEC DBMS_SERVICE.START_SERVICE('tc_service');

-- Verify
SELECT NAME, TRUE_CACHE_SERVICE, NETWORK_NAME
FROM   DBA_SERVICES
WHERE  TRUE_CACHE_SERVICE IS NOT NULL;
```

---

### Step 13 — Auto-start service after restarts

```sql
CREATE OR REPLACE TRIGGER start_tc_service
  AFTER STARTUP ON DATABASE
BEGIN
  DBMS_SERVICE.START_SERVICE('tc_service');
END;
/
```

---

## Phase 4 — Application Connection

> Zero application code changes required. The Oracle JDBC thin driver (26ai) automatically detects read-only transactions and routes them to True Cache.

---

### Java — JDBC thin driver

```java
// Connect to True Cache service
String url = "jdbc:oracle:thin:@truecache-host:1521/tc_service";

Connection conn = DriverManager.getConnection(
    url, "appuser", "apppassword");

// Read-only transactions  → auto-routed to True Cache
// Write transactions       → auto-routed to Primary
```

---

### Java — Universal Connection Pool (UCP)

```java
PoolDataSource pds = PoolDataSourceFactory.getPoolDataSource();
pds.setConnectionFactoryClassName(
    "oracle.jdbc.pool.OracleDataSource");
pds.setURL("jdbc:oracle:thin:@truecache-host:1521/tc_service");
pds.setUser("appuser");
pds.setPassword("apppassword");
pds.setInitialPoolSize(5);
pds.setMaxPoolSize(20);

// UCP auto-routes:
//   Read-only transactions  → True Cache
//   Write transactions      → Primary
```

---

### Python — python-oracledb

```python
import oracledb

conn = oracledb.connect(
    user='appuser',
    password='apppassword',
    dsn='truecache-host:1521/tc_service'
)

cursor = conn.cursor()
cursor.execute(
    'SELECT * FROM customers WHERE region = :r',
    r='APAC'
)  # Served from True Cache buffer cache
```

---

### Force read-only routing explicitly

```sql
-- Pin a session to True Cache
ALTER SESSION SET TRUE_CACHE_SERVICE = 'tc_service';

-- Statement-level hint
SELECT /*+ TRUE_CACHE */ *
FROM   orders
WHERE  status = 'SHIPPED';
```

---

## Phase 5 — Monitoring & Management

---

### True Cache hit ratio

> Target: **> 80%**. Low hit ratio = SGA too small → increase `SGA_TARGET`.

```sql
-- Run on TRUE CACHE instance
SELECT METRIC_NAME, VALUE
FROM   V$TRUE_CACHE_STATS
ORDER  BY METRIC_NAME;

-- Key metrics:
--   TRUE_CACHE_HIT_RATIO        (target > 80%)
--   BLOCKS_SERVED_FROM_CACHE
--   BLOCKS_FETCHED_FROM_PRIMARY
--   TOTAL_QUERIES_SERVED
```

---

### Apply lag

```sql
-- Run on PRIMARY
SELECT NAME, VALUE, UNIT, TIME_COMPUTED
FROM   V$DATAGUARD_STATS
WHERE  NAME IN ('apply lag','transport lag')
AND    DB_UNIQUE_NAME = 'TRUECACHE1';

-- Run on TRUE CACHE
SELECT CURRENT_SCN, APPLIED_SCN,
       (CURRENT_SCN - APPLIED_SCN) AS SCN_LAG
FROM   V$DATABASE;
```

---

### Buffer cache pressure

```sql
-- Buffer cache hit rate on True Cache
SELECT 1 - (SUM(physical_reads) /
           (SUM(db_block_gets) + SUM(consistent_gets)))
         AS buffer_hit_ratio
FROM   V$BUFFER_POOL_STATISTICS;

-- Resize SGA on the fly
ALTER SYSTEM SET SGA_TARGET=16G SCOPE=BOTH;
```

---

### Lifecycle commands

| Action | Command |
|---|---|
| Stop True Cache | `SHUTDOWN IMMEDIATE;` (on True Cache host) |
| Start True Cache | `STARTUP;` (on True Cache host) |
| Check status | `SELECT STATUS, DATABASE_ROLE FROM V$DATABASE;` |
| Suspend redo apply | `ALTER DATABASE RECOVER MANAGED STANDBY DATABASE CANCEL;` |
| Resume redo apply | `ALTER DATABASE RECOVER MANAGED STANDBY DATABASE DISCONNECT;` |
| Remove True Cache | `dbca -silent -deleteTrueCache -sid TRUECACHE1 ...` |
| Disable dest on primary | `ALTER SYSTEM SET LOG_ARCHIVE_DEST_STATE_2=DEFER SCOPE=BOTH;` |

---

## True Cache vs Standby — Step-by-Step Comparison

> If you have built a Physical Standby before, you already know **60% of True Cache setup**. The primary-side configuration is identical. True Cache removes the three most time-consuming standby tasks.

### Legend
- ✅ **Identical** — same step, same command
- 🔶 **Different** — similar concept, different detail
- ❌ **Not needed** — True Cache skips this entirely
- 🔵 **TC only** — True Cache specific, no standby equivalent

---

### On PRIMARY — prepare the source database

| # | Step | Match | Active Data Guard Standby | True Cache |
|---|---|:---:|---|---|
| 1 | ARCHIVELOG mode | ✅ | `ALTER DATABASE ARCHIVELOG;` | Same command |
| 2 | FORCE LOGGING | ✅ | `ALTER DATABASE FORCE LOGGING;` | Same command |
| 3 | DB_UNIQUE_NAME | ✅ | Set in SPFILE e.g. `PRODDB` | Same parameter |
| 4 | Standby redo logs on primary | 🔶 | Required for fast failover | **Not required** |
| 5 | LOG_ARCHIVE_DEST_n | ✅ | `SERVICE=` pointing to standby | Same syntax — `SERVICE=` pointing to True Cache |
| 6 | LOG_ARCHIVE_CONFIG DG_CONFIG | ✅ | `DG_CONFIG=(PRODDB,STANDBY1)` | `DG_CONFIG=(PRODDB,TRUECACHE1)` — same syntax |
| 7 | Password file | ✅ | `orapwd` on primary, `scp` to standby | Same — `scp` orapw file to True Cache host |
| 8 | TNS entries | ✅ | Add standby alias in `tnsnames.ora` | Add True Cache alias — same format |

---

### On STANDBY / TRUE CACHE HOST — create the instance

| # | Step | Match | Active Data Guard Standby | True Cache |
|---|---|:---:|---|---|
| 9 | Oracle software install | ✅ | Same version + patches as primary | Same |
| 10 | RMAN backup / restore | ❌ | **Required** — `RMAN DUPLICATE FROM ACTIVE DATABASE` to seed datafiles | **Not needed** — no datafiles, buffer cache builds lazily |
| 11 | Standby redo logs on standby | ❌ | **Required** — match primary log size, count +1 group per thread | **Not needed** — skipped entirely |
| 12 | init.ora / SPFILE | 🔶 | Standard params: `DB_UNIQUE_NAME`, `FAL_SERVER`, `DB_FILE_NAME_CONVERT` | Must include `TRUE_CACHE=TRUE`. No `DB_FILE_NAME_CONVERT`. Focus on `SGA_TARGET`. |
| 13 | Instance creation | 🔶 | `STARTUP NOMOUNT` + `RMAN DUPLICATE`, or DBCA duplicate wizard | `dbca -createTrueCache` — one command, no RMAN |
| 14 | Redo Apply startup | 🔶 | `ALTER DATABASE RECOVER MANAGED STANDBY DATABASE DISCONNECT FROM SESSION;` | Started automatically by DBCA — no manual command |
| 15 | Open database | 🔶 | `ALTER DATABASE OPEN READ ONLY;` then start MRP | DBCA handles this — opens automatically in `READ ONLY WITH APPLY` |

---

### Service & application configuration

| # | Step | Match | Active Data Guard Standby | True Cache |
|---|---|:---:|---|---|
| 16 | Read service creation | 🔶 | `DBMS_SERVICE.CREATE_SERVICE` with standard params. App must use standby URL explicitly. | `DBMS_SERVICE.CREATE_SERVICE` with `TRUE_CACHE_SERVICE` param. JDBC auto-routes. |
| 17 | Application changes required | 🔶 | Separate connection pool or URL for standby reads. Code change often needed. | **Zero application changes.** JDBC thin driver auto-routes read-only transactions. |
| 18 | Hit ratio monitoring | 🔵 | Not applicable | Monitor `V$TRUE_CACHE_STATS` — `TRUE_CACHE_HIT_RATIO` target > 80% |

---

### Ongoing management

| # | Step | Match | Active Data Guard Standby | True Cache |
|---|---|:---:|---|---|
| 19 | Failover / switchover | 🔶 | Full Data Guard failover and switchover. Can become primary. | No failover. Read-offload only. Cannot become primary. |
| 20 | Disk storage | 🔶 | Full ASM disk group — same size as primary. Monitor space, RMAN backups. | **No disk.** Monitor SGA/buffer cache hit ratio only. |
| 21 | RMAN backups | 🔶 | Common pattern — offload RMAN to standby to reduce primary I/O. | Not applicable — True Cache holds no persistent data. |
| 22 | Apply lag monitoring | ✅ | `V$DATAGUARD_STATS` — apply lag, transport lag | Same views, same alert thresholds |

---

### What True Cache removes vs a Physical Standby

| Physical Standby requires | True Cache — this step is gone |
|---|---|
| RMAN DUPLICATE to seed all datafiles | ✅ No RMAN step — buffer cache builds lazily |
| Standby redo log creation on both hosts | ✅ No standby redo logs on either host |
| ASM disk group sizing and provisioning | ✅ No disk storage to provision or manage |
| `STARTUP NOMOUNT` / `MOUNT` sequence | ✅ `dbca -createTrueCache` does everything |
| MRP process management | ✅ Apply starts automatically |
| Separate app connection URL / pool for reads | ✅ JDBC auto-routes — zero app changes |
| RMAN backup jobs on standby | ✅ No backups — True Cache is stateless |

---

## Setup Checklist

Use this checklist to track progress. All 13 steps must be completed in order.

### Phase 1 — Primary database
- [ ] Step 1 — Verify / enable ARCHIVELOG mode
- [ ] Step 2 — Enable FORCE LOGGING
- [ ] Step 3 — Set DB_UNIQUE_NAME
- [ ] Step 4 — Configure LOG_ARCHIVE_DEST_2 for True Cache
- [ ] Step 5 — Create and distribute password file
- [ ] Step 6 — Add True Cache TNS entry on primary

### Phase 2 — True Cache host
- [ ] Step 7 — Install Oracle AI Database 26ai software
- [ ] Step 8 — Create seed init.ora with `TRUE_CACHE=TRUE`
- [ ] Step 9 — Add primary TNS entry on True Cache host
- [ ] Step 10 — Run `dbca -createTrueCache`
- [ ] Step 11 — Verify STATUS and apply lag

### Phase 3 — Service configuration
- [ ] Step 12 — Create and start `tc_service` via `DBMS_SERVICE`
- [ ] Step 13 — Create auto-start trigger

### Phase 4 — Application
- [ ] Connect app to `truecache-host:1521/tc_service`
- [ ] Verify JDBC auto-routing (check `V$TRUE_CACHE_STATS`)

---

## Quick Reference

```sql
-- Is True Cache running?
SELECT STATUS, DATABASE_ROLE, OPEN_MODE FROM V$DATABASE;

-- Apply lag
SELECT NAME, VALUE, UNIT FROM V$DATAGUARD_STATS
WHERE NAME IN ('apply lag','transport lag');

-- Hit ratio (run on True Cache)
SELECT METRIC_NAME, VALUE FROM V$TRUE_CACHE_STATS;

-- Services
SELECT NAME, TRUE_CACHE_SERVICE FROM DBA_SERVICES
WHERE TRUE_CACHE_SERVICE IS NOT NULL;
```

---

## Key takeaway

> If you have built a Physical Standby before, the primary-side configuration is **identical**.
> True Cache removes the three most time-consuming standby tasks:
> **RMAN restore · Standby redo log creation · Manual MRP startup**
> — replaced by a single `dbca -createTrueCache` command.

---

*Oracle AI Database 26ai — True Cache DBA Reference · v26ai.1 · March 2026*
