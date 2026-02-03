# Database Profiling

Query analysis, lock contention, buffer management, and storage engine internals.

## PostgreSQL

### Query Profiling

#### EXPLAIN ANALYZE

The primary tool for understanding query execution. Shows actual vs estimated costs.

```sql
-- Basic execution plan
EXPLAIN SELECT * FROM users WHERE email = 'user@example.com';

-- With actual execution (runs the query)
EXPLAIN ANALYZE SELECT * FROM users WHERE email = 'user@example.com';

-- Full detail: buffers, timing, WAL
EXPLAIN (ANALYZE, BUFFERS, TIMING, WAL)
SELECT * FROM orders WHERE created_at > now() - interval '1 day';

-- Output formats
EXPLAIN (FORMAT JSON) SELECT ...;
EXPLAIN (FORMAT YAML) SELECT ...;
EXPLAIN (FORMAT TEXT) SELECT ...;  -- default
```

**Key metrics in output:**
| Field | Meaning |
|-------|---------|
| `cost=0.00..123.45` | Startup cost..total cost (arbitrary units) |
| `rows=1000` | Estimated rows returned |
| `actual time=0.015..45.678` | Actual execution time (ms) |
| `loops=1` | Number of times node executed |
| `Buffers: shared hit=123` | Pages read from cache |
| `Buffers: shared read=45` | Pages read from disk |

**Common plan nodes:**
```
Seq Scan          -- Full table scan, check for missing index
Index Scan        -- B-tree index lookup
Index Only Scan   -- Covered query, no heap access needed
Bitmap Heap Scan  -- Multiple index conditions combined
Nested Loop       -- For each outer row, scan inner
Hash Join         -- Build hash table, probe with other side
Merge Join        -- Both sides sorted, merge
Sort              -- In-memory or on-disk sort
```

#### pg_stat_statements

Aggregates query statistics across all executions. Essential for finding expensive queries.

```sql
-- Enable extension
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;

-- Top queries by total time
SELECT
    substring(query, 1, 80) as query,
    calls,
    round(total_exec_time::numeric, 2) as total_ms,
    round(mean_exec_time::numeric, 2) as mean_ms,
    round((100 * total_exec_time / sum(total_exec_time) OVER ())::numeric, 2) as pct
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 20;

-- Queries with high I/O
SELECT
    substring(query, 1, 60) as query,
    calls,
    shared_blks_hit + shared_blks_read as total_blks,
    round(100.0 * shared_blks_hit / nullif(shared_blks_hit + shared_blks_read, 0), 2) as hit_pct
FROM pg_stat_statements
WHERE shared_blks_read > 1000
ORDER BY shared_blks_read DESC
LIMIT 20;

-- Reset statistics
SELECT pg_stat_statements_reset();
```

**Configuration (postgresql.conf):**
```
shared_preload_libraries = 'pg_stat_statements'
pg_stat_statements.max = 10000
pg_stat_statements.track = all        # none, top, all
pg_stat_statements.track_utility = on # track non-DML
```

#### auto_explain

Automatically logs slow query plans. Useful for production diagnosis.

```sql
-- Enable for session
LOAD 'auto_explain';
SET auto_explain.log_min_duration = '100ms';
SET auto_explain.log_analyze = on;
SET auto_explain.log_buffers = on;

-- Or in postgresql.conf for all sessions:
-- shared_preload_libraries = 'auto_explain'
-- auto_explain.log_min_duration = '1s'
-- auto_explain.log_analyze = on
```

### Connection Analysis

#### pg_stat_activity

Shows all current connections and their state.

```sql
-- Current connections overview
SELECT
    state,
    count(*) as count,
    max(now() - state_change) as max_duration
FROM pg_stat_activity
WHERE backend_type = 'client backend'
GROUP BY state;

-- Long-running queries
SELECT
    pid,
    now() - query_start as duration,
    state,
    wait_event_type,
    wait_event,
    substring(query, 1, 80) as query
FROM pg_stat_activity
WHERE state != 'idle'
  AND query_start < now() - interval '30 seconds'
ORDER BY query_start;

-- Blocked queries
SELECT
    blocked.pid as blocked_pid,
    blocked.query as blocked_query,
    blocking.pid as blocking_pid,
    blocking.query as blocking_query
FROM pg_stat_activity blocked
JOIN pg_stat_activity blocking ON blocking.pid = ANY(pg_blocking_pids(blocked.pid))
WHERE blocked.state = 'active';

-- Kill a query
SELECT pg_cancel_backend(pid);    -- graceful (SIGINT)
SELECT pg_terminate_backend(pid); -- force (SIGTERM)
```

#### Connection Pooling (PgBouncer)

PgBouncer reduces connection overhead. PostgreSQL connections are expensive (~5-10MB each).

```bash
# PgBouncer stats
psql -p 6432 -U pgbouncer pgbouncer -c "SHOW STATS;"
psql -p 6432 -U pgbouncer pgbouncer -c "SHOW POOLS;"
psql -p 6432 -U pgbouncer pgbouncer -c "SHOW CLIENTS;"
psql -p 6432 -U pgbouncer pgbouncer -c "SHOW SERVERS;"
```

**Key metrics:**
| Metric | Description |
|--------|-------------|
| `cl_active` | Clients running queries |
| `cl_waiting` | Clients waiting for connection |
| `sv_active` | Server connections in use |
| `sv_idle` | Server connections available |
| `avg_query_time` | Average query time (microseconds) |

**pgbouncer.ini tuning:**
```ini
[databases]
mydb = host=localhost dbname=mydb

[pgbouncer]
pool_mode = transaction      # session, transaction, statement
max_client_conn = 1000
default_pool_size = 25
min_pool_size = 5
reserve_pool_size = 5
reserve_pool_timeout = 3
server_idle_timeout = 60
```

#### Idle Connection Impact

Idle connections consume memory and affect autovacuum worker allocation.

```sql
-- Count idle connections per database
SELECT datname, state, count(*)
FROM pg_stat_activity
GROUP BY datname, state
ORDER BY datname, state;

-- Set connection timeout (postgresql.conf)
-- idle_in_transaction_session_timeout = '5min'
-- idle_session_timeout = '30min'  -- PostgreSQL 14+
```

### Index Efficiency

#### pg_stat_user_indexes

Identifies unused or inefficient indexes.

```sql
-- Unused indexes (candidates for removal)
SELECT
    schemaname || '.' || relname as table,
    indexrelname as index,
    pg_size_pretty(pg_relation_size(indexrelid)) as size,
    idx_scan as scans
FROM pg_stat_user_indexes
WHERE idx_scan = 0
  AND indexrelid NOT IN (SELECT conindid FROM pg_constraint)
ORDER BY pg_relation_size(indexrelid) DESC;

-- Index usage ratio
SELECT
    schemaname || '.' || relname as table,
    seq_scan,
    idx_scan,
    round(100.0 * idx_scan / nullif(seq_scan + idx_scan, 0), 2) as idx_pct
FROM pg_stat_user_tables
WHERE seq_scan + idx_scan > 100
ORDER BY seq_scan DESC;

-- Index size vs table size
SELECT
    t.schemaname || '.' || t.relname as table,
    pg_size_pretty(pg_relation_size(t.relid)) as table_size,
    pg_size_pretty(sum(pg_relation_size(i.indexrelid))) as index_size,
    round(100.0 * sum(pg_relation_size(i.indexrelid)) /
          nullif(pg_relation_size(t.relid), 0), 2) as ratio
FROM pg_stat_user_tables t
LEFT JOIN pg_stat_user_indexes i ON t.relid = i.relid
GROUP BY t.schemaname, t.relname, t.relid
ORDER BY pg_relation_size(t.relid) DESC;
```

#### Index-Only Scans

Most efficient scan type - reads only index, no heap access.

```sql
-- Check if index-only scans are working
SELECT
    indexrelname,
    idx_scan,
    idx_tup_read,
    idx_tup_fetch  -- should be 0 for pure index-only scans
FROM pg_stat_user_indexes
WHERE idx_scan > 0
ORDER BY idx_tup_fetch DESC;

-- Visibility map must be updated for index-only scans
-- Run VACUUM to update visibility map
VACUUM (VERBOSE) tablename;
```

#### Bloat Detection

Table and index bloat wastes space and slows queries.

```sql
-- Estimate table bloat (pgstattuple extension)
CREATE EXTENSION IF NOT EXISTS pgstattuple;
SELECT * FROM pgstattuple('tablename');

-- Simple bloat estimate query
SELECT
    schemaname || '.' || relname as table,
    pg_size_pretty(pg_relation_size(relid)) as size,
    n_dead_tup,
    round(100.0 * n_dead_tup / nullif(n_live_tup + n_dead_tup, 0), 2) as dead_pct,
    last_vacuum,
    last_autovacuum
FROM pg_stat_user_tables
WHERE n_dead_tup > 1000
ORDER BY n_dead_tup DESC;

-- Repack to reclaim space (pg_repack extension)
-- pg_repack --table tablename dbname
```

### Lock Analysis

#### pg_locks

View current locks and detect contention.

```sql
-- Current locks
SELECT
    l.locktype,
    l.relation::regclass,
    l.mode,
    l.granted,
    a.pid,
    a.query
FROM pg_locks l
JOIN pg_stat_activity a ON l.pid = a.pid
WHERE l.relation IS NOT NULL
ORDER BY l.relation;

-- Lock conflicts
SELECT
    blocked.pid as blocked_pid,
    blocked.query as blocked_query,
    blocking.pid as blocking_pid,
    blocking.query as blocking_query,
    blocked_locks.mode as blocked_mode,
    blocking_locks.mode as blocking_mode
FROM pg_locks blocked_locks
JOIN pg_stat_activity blocked ON blocked.pid = blocked_locks.pid
JOIN pg_locks blocking_locks ON blocking_locks.locktype = blocked_locks.locktype
    AND blocking_locks.relation = blocked_locks.relation
    AND blocking_locks.pid != blocked_locks.pid
JOIN pg_stat_activity blocking ON blocking.pid = blocking_locks.pid
WHERE NOT blocked_locks.granted;
```

#### Deadlock Detection

PostgreSQL automatically detects and breaks deadlocks.

```sql
-- Check deadlock count
SELECT deadlocks FROM pg_stat_database WHERE datname = current_database();

-- Enable deadlock logging (postgresql.conf)
-- log_lock_waits = on
-- deadlock_timeout = 1s
```

```bash
# Search logs for deadlocks
grep -i "deadlock detected" /var/log/postgresql/*.log
```

#### Lock Wait Events

Use pg_stat_activity wait events to identify contention.

```sql
-- Wait event summary
SELECT
    wait_event_type,
    wait_event,
    count(*)
FROM pg_stat_activity
WHERE wait_event IS NOT NULL
GROUP BY wait_event_type, wait_event
ORDER BY count DESC;

-- Common wait events:
-- Lock:relation         - waiting for table lock
-- Lock:tuple           - waiting for row lock
-- LWLock:buffer_content - buffer pool contention
-- IO:DataFileRead      - reading data from disk
```

## MySQL

### Slow Query Analysis

#### slow_query_log

Logs queries exceeding threshold. First step in optimization.

```sql
-- Enable slow query log
SET GLOBAL slow_query_log = 'ON';
SET GLOBAL long_query_time = 1;           -- seconds
SET GLOBAL log_queries_not_using_indexes = 'ON';
SET GLOBAL slow_query_log_file = '/var/log/mysql/slow.log';

-- Check current settings
SHOW VARIABLES LIKE 'slow_query%';
SHOW VARIABLES LIKE 'long_query_time';
```

**my.cnf configuration:**
```ini
[mysqld]
slow_query_log = 1
slow_query_log_file = /var/log/mysql/slow.log
long_query_time = 1
log_queries_not_using_indexes = 1
min_examined_row_limit = 100
```

#### mysqldumpslow

Aggregates and summarizes slow query log.

```bash
# Top 10 by count
mysqldumpslow -s c -t 10 /var/log/mysql/slow.log

# Top 10 by time
mysqldumpslow -s t -t 10 /var/log/mysql/slow.log

# Sort options: c (count), t (time), l (lock time), r (rows)
# Additional: at, al, ar for averages

# Filter by pattern
mysqldumpslow -s t -t 10 -g "SELECT" /var/log/mysql/slow.log
```

#### pt-query-digest

Percona toolkit - more powerful than mysqldumpslow.

```bash
# Basic analysis
pt-query-digest /var/log/mysql/slow.log

# Filter by time range
pt-query-digest --since '2024-01-01' --until '2024-01-02' slow.log

# Only specific database
pt-query-digest --filter '$event->{db} eq "mydb"' slow.log

# Output to file
pt-query-digest --output=report slow.log > report.txt

# From tcpdump (live capture)
tcpdump -s 65535 -x -nn -q -tttt -i any port 3306 > mysql.tcp
pt-query-digest --type tcpdump mysql.tcp
```

### Performance Schema

MySQL's built-in instrumentation framework. More detailed than slow query log.

#### Key Tables

```sql
-- Enable all statement instrumentation
UPDATE performance_schema.setup_instruments
SET ENABLED = 'YES', TIMED = 'YES'
WHERE NAME LIKE 'statement/%';

UPDATE performance_schema.setup_consumers
SET ENABLED = 'YES'
WHERE NAME LIKE 'events_statements%';

-- Top queries by total time
SELECT
    DIGEST_TEXT,
    COUNT_STAR as calls,
    ROUND(SUM_TIMER_WAIT/1000000000000, 3) as total_sec,
    ROUND(AVG_TIMER_WAIT/1000000000, 3) as avg_ms,
    SUM_ROWS_EXAMINED,
    SUM_ROWS_SENT
FROM performance_schema.events_statements_summary_by_digest
ORDER BY SUM_TIMER_WAIT DESC
LIMIT 20;

-- Current queries
SELECT * FROM performance_schema.events_statements_current
WHERE SQL_TEXT IS NOT NULL;

-- Statement history
SELECT * FROM performance_schema.events_statements_history
ORDER BY TIMER_START DESC LIMIT 20;
```

#### Wait Event Analysis

```sql
-- Enable wait instrumentation
UPDATE performance_schema.setup_instruments
SET ENABLED = 'YES', TIMED = 'YES'
WHERE NAME LIKE 'wait/%';

-- Top wait events
SELECT
    EVENT_NAME,
    COUNT_STAR,
    ROUND(SUM_TIMER_WAIT/1000000000, 2) as total_ms
FROM performance_schema.events_waits_summary_global_by_event_name
WHERE COUNT_STAR > 0
ORDER BY SUM_TIMER_WAIT DESC
LIMIT 20;

-- Wait events by thread
SELECT
    t.THREAD_ID,
    t.PROCESSLIST_USER,
    t.PROCESSLIST_DB,
    w.EVENT_NAME,
    w.TIMER_WAIT/1000000 as wait_us
FROM performance_schema.threads t
JOIN performance_schema.events_waits_current w ON t.THREAD_ID = w.THREAD_ID
WHERE w.EVENT_NAME NOT LIKE 'idle%';
```

#### Statement Analysis

```sql
-- Queries causing most disk reads
SELECT
    DIGEST_TEXT,
    COUNT_STAR,
    SUM_ROWS_EXAMINED,
    SUM_NO_INDEX_USED + SUM_NO_GOOD_INDEX_USED as no_index
FROM performance_schema.events_statements_summary_by_digest
WHERE SUM_ROWS_EXAMINED > 10000
ORDER BY SUM_ROWS_EXAMINED DESC
LIMIT 10;

-- Tables with most I/O
SELECT
    OBJECT_SCHEMA,
    OBJECT_NAME,
    COUNT_READ,
    COUNT_WRITE,
    SUM_TIMER_READ/1000000000 as read_sec,
    SUM_TIMER_WRITE/1000000000 as write_sec
FROM performance_schema.table_io_waits_summary_by_table
ORDER BY SUM_TIMER_WAIT DESC
LIMIT 20;
```

### InnoDB Tuning

#### Buffer Pool Monitoring

The buffer pool caches data and indexes. Most critical InnoDB setting.

```sql
-- Buffer pool status
SHOW ENGINE INNODB STATUS\G

-- Buffer pool metrics
SELECT
    POOL_ID,
    POOL_SIZE,
    FREE_BUFFERS,
    DATABASE_PAGES,
    PAGES_MADE_YOUNG,
    PAGES_NOT_MADE_YOUNG,
    HIT_RATE
FROM information_schema.INNODB_BUFFER_POOL_STATS;

-- Buffer pool hit ratio (should be >99%)
SHOW GLOBAL STATUS LIKE 'Innodb_buffer_pool%';

-- Calculate hit ratio
SELECT
    (1 - (
        (SELECT VARIABLE_VALUE FROM performance_schema.global_status WHERE VARIABLE_NAME = 'Innodb_buffer_pool_reads') /
        (SELECT VARIABLE_VALUE FROM performance_schema.global_status WHERE VARIABLE_NAME = 'Innodb_buffer_pool_read_requests')
    )) * 100 as hit_ratio;
```

**Configuration:**
```ini
[mysqld]
innodb_buffer_pool_size = 12G      # 70-80% of RAM for dedicated server
innodb_buffer_pool_instances = 8   # 1 per GB up to 8
innodb_log_file_size = 2G          # Larger = better write performance
innodb_flush_log_at_trx_commit = 1 # 1=durable, 2=faster
innodb_flush_method = O_DIRECT     # Avoid double buffering
```

#### Adaptive Hash Index

InnoDB automatically builds hash indexes for frequently accessed pages.

```sql
-- AHI status
SHOW ENGINE INNODB STATUS\G
-- Look for "INSERT BUFFER AND ADAPTIVE HASH INDEX" section

-- AHI metrics
SHOW GLOBAL STATUS LIKE 'Innodb_adaptive_hash%';

-- Disable if causing contention (rare)
-- innodb_adaptive_hash_index = OFF
```

#### Change Buffer

Caches secondary index changes to reduce random I/O.

```sql
-- Change buffer metrics
SHOW GLOBAL STATUS LIKE 'Innodb_ibuf%';

-- Change buffer settings
SHOW VARIABLES LIKE 'innodb_change_buffer%';

-- Tune max size (% of buffer pool)
-- innodb_change_buffer_max_size = 25  -- default 25%
```

## LSM-Tree Databases (RocksDB)

RocksDB is used by MySQL (MyRocks), TiKV, CockroachDB, and others. Optimized for write-heavy workloads.

### Compaction Profiling

LSM-trees require compaction to merge sorted runs. Major source of I/O and CPU usage.

```bash
# RocksDB statistics (via LOG file or programmatic access)
# Key metrics in LOG:
grep "Compaction" LOG | tail -20
grep "compaction_stats" LOG
```

**Key compaction metrics:**
| Metric | Description |
|--------|-------------|
| `rocksdb.compact.read.bytes` | Bytes read during compaction |
| `rocksdb.compact.write.bytes` | Bytes written during compaction |
| `rocksdb.compaction.times.micros` | Time spent compacting |
| `rocksdb.num.running.compactions` | Currently running compactions |

```sql
-- MyRocks compaction stats (MySQL with RocksDB)
SELECT * FROM information_schema.ROCKSDB_COMPACTION_STATS;
SELECT * FROM information_schema.ROCKSDB_CFSTATS;
```

### Write Amplification

Write amplification = total bytes written to storage / bytes written by application.

```bash
# Calculate write amplification
# From RocksDB statistics:
# WA = (compaction_bytes_written + flush_bytes_written) / user_bytes_written

# Ideal: 10-30x for typical workloads
# High WA (>50x) indicates tuning needed
```

**Reducing write amplification:**
```
# Increase level size ratio (fewer levels)
level_compaction_dynamic_level_bytes = true
max_bytes_for_level_multiplier = 10

# Use universal compaction for write-heavy
compaction_style = universal

# Increase memtable size (fewer flushes)
write_buffer_size = 256MB
max_write_buffer_number = 4
```

### Level Analysis

```bash
# View level statistics
# RocksDB LOG shows per-level stats:
# Level  Files Size(MB)  Score Read(GB) Rn(GB) Rnp1(GB) Write(GB)
#   L0    4/0      64     1.0     0.0    0.0      0.0       0.1
#   L1    5/0     256     1.0     0.2    0.1      0.1       0.2
```

```sql
-- MyRocks level info
SHOW ENGINE ROCKSDB STATUS\G
-- Look for "Level" section
```

### RocksDB Tuning

#### Block Cache

In-memory cache for data blocks. Analogous to InnoDB buffer pool.

```
# Set block cache size (typically 30-50% of RAM)
block_cache_size = 8GB

# Enable compressed block cache for larger datasets
block_cache_compressed = true

# Cache index and filter blocks
cache_index_and_filter_blocks = true
pin_l0_filter_and_index_blocks_in_cache = true
```

#### Bloom Filters

Bloom filters reduce disk reads for non-existent keys.

```
# Enable bloom filters
filter_policy = bloomfilter:10:false
# 10 bits per key, ~1% false positive rate

# For prefix scans
prefix_extractor = fixed:8
# Use with bloom filters for prefix queries

# Whole key filtering for point lookups
whole_key_filtering = true
```

#### Compression

Balance between CPU and I/O. Different per level is common.

```
# Per-level compression
compression_per_level = no:no:lz4:lz4:lz4:zstd:zstd

# L0-L1: no compression (hot data)
# L2-L4: lz4 (fast, moderate ratio)
# L5+: zstd (better ratio, more CPU)

# Compression options
compression_opts = -14:32767:0:4096
# level:window_bits:strategy:max_dict_bytes
```

## General Patterns

### Index Strategy

#### Covering Indexes

Include all columns needed by query to avoid heap/table access.

```sql
-- PostgreSQL
CREATE INDEX idx_orders_covering ON orders (customer_id)
    INCLUDE (order_date, total);

-- MySQL
CREATE INDEX idx_orders_covering ON orders (customer_id, order_date, total);

-- Verify index-only access
EXPLAIN SELECT order_date, total FROM orders WHERE customer_id = 123;
-- Look for: "Index Only Scan" (PG) or "Using index" (MySQL)
```

#### Partial Indexes

Index only rows matching a condition. Smaller, faster.

```sql
-- PostgreSQL only
CREATE INDEX idx_orders_pending ON orders (created_at)
    WHERE status = 'pending';

-- Orders only for recent active users
CREATE INDEX idx_recent_active ON orders (user_id, created_at)
    WHERE created_at > '2024-01-01';

-- Significantly smaller than full index
SELECT pg_size_pretty(pg_relation_size('idx_orders_pending'));
```

#### Index Maintenance

```sql
-- PostgreSQL: Reindex to fix bloat
REINDEX INDEX CONCURRENTLY idx_name;
REINDEX TABLE CONCURRENTLY tablename;

-- MySQL: Optimize table (rebuilds indexes)
OPTIMIZE TABLE tablename;
-- Or use pt-online-schema-change for large tables

-- Analyze for fresh statistics
ANALYZE tablename;          -- PostgreSQL
ANALYZE TABLE tablename;    -- MySQL
```

### Buffer/Cache Analysis

#### Hit Rates

```sql
-- PostgreSQL buffer cache hit ratio
SELECT
    sum(blks_hit) as hits,
    sum(blks_read) as reads,
    round(100.0 * sum(blks_hit) / nullif(sum(blks_hit) + sum(blks_read), 0), 2) as hit_ratio
FROM pg_stat_database;

-- Per-table cache usage (pg_buffercache extension)
CREATE EXTENSION IF NOT EXISTS pg_buffercache;
SELECT
    c.relname,
    count(*) as buffers,
    pg_size_pretty(count(*) * 8192) as cached
FROM pg_buffercache b
JOIN pg_class c ON b.relfilenode = pg_relation_filenode(c.oid)
WHERE c.relname NOT LIKE 'pg_%'
GROUP BY c.relname
ORDER BY count(*) DESC
LIMIT 20;
```

```sql
-- MySQL InnoDB buffer pool contents
SELECT
    table_name,
    index_name,
    count(*) as pages,
    sum(data_size)/1024/1024 as data_mb
FROM information_schema.INNODB_BUFFER_PAGE
WHERE table_name IS NOT NULL
GROUP BY table_name, index_name
ORDER BY pages DESC
LIMIT 20;
```

#### Eviction Patterns

```sql
-- PostgreSQL: Track evictions via pg_stat_bgwriter
SELECT
    buffers_clean,          -- Cleaned by background writer
    buffers_backend,        -- Cleaned by backends (bad - indicates undersized)
    buffers_alloc           -- Total allocations
FROM pg_stat_bgwriter;

-- High buffers_backend indicates buffer pool too small
```

```sql
-- MySQL InnoDB eviction stats
SHOW GLOBAL STATUS LIKE 'Innodb_buffer_pool_pages%';
SHOW GLOBAL STATUS LIKE 'Innodb_buffer_pool_read%';

-- pages_flushed increasing rapidly = memory pressure
-- read_ahead_evicted = prefetched pages evicted before use
```

#### Memory Pressure

```sql
-- PostgreSQL shared buffers utilization
SELECT
    count(*) as total_buffers,
    count(*) FILTER (WHERE usagecount > 0) as in_use,
    round(100.0 * count(*) FILTER (WHERE usagecount > 0) / count(*), 2) as usage_pct
FROM pg_buffercache;

-- OS-level memory pressure
-- Check: cat /proc/meminfo | grep -E "MemFree|Cached|Buffers"
```

```bash
# Database-specific memory monitoring
# PostgreSQL
ps aux | grep postgres | awk '{sum+=$6} END {print sum/1024 " MB"}'

# MySQL
mysqladmin -u root -p extended-status | grep -i buffer

# Generic: memory by process
smem -tk -c "pid user command swap uss pss rss" | grep -E "postgres|mysql"
```

## Quick Reference

| Task | PostgreSQL | MySQL |
|------|------------|-------|
| Query plan | `EXPLAIN ANALYZE` | `EXPLAIN ANALYZE` |
| Top queries | `pg_stat_statements` | `performance_schema.events_statements_summary_by_digest` |
| Current queries | `pg_stat_activity` | `SHOW PROCESSLIST` |
| Kill query | `pg_cancel_backend(pid)` | `KILL QUERY id` |
| Lock info | `pg_locks` | `information_schema.INNODB_LOCKS` |
| Buffer stats | `pg_buffercache` | `INNODB_BUFFER_POOL_STATS` |
| Index usage | `pg_stat_user_indexes` | `performance_schema.table_io_waits_summary_by_index_usage` |
| Slow log | `auto_explain` | `slow_query_log` |
| Deadlocks | `pg_stat_database.deadlocks` | `SHOW ENGINE INNODB STATUS` |
| Vacuum/Optimize | `VACUUM ANALYZE` | `OPTIMIZE TABLE` |
