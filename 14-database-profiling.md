# Database Profiling

Query analysis, lock contention, buffer management, and storage engine internals.

**See also:** [Database Production Debugging](database-production-debugging.md) for hot partition rate limiting, prepared statement cache pollution, memory-aware admission control, and victim vs culprit query identification.

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

## MySQL High Availability

> Source: [GitHub - MySQL High Availability at GitHub](https://github.blog/2018-06-20-mysql-high-availability-at-github/)

### Failover Stack

```
orchestrator (Raft consensus)
     ↓  promotes replica, updates primary identity
Consul KV  (primary identity store)
     ↓  watched by load balancer
GLB/HAProxy (Consul-backed backend pools, anycast)
     ↓
Application (connects via VIP — primary identity hidden)
```

**Why this works:**
- Raft consensus prevents split-brain: an isolated DC cannot form quorum → cannot become leader → no conflicting failovers
- Promotion + Consul update happen concurrently — primary accepts writes before replication tree is fully repaired (non-blocking)
- HAProxy `hard-stop-after` cleans stale connections automatically on backend change
- GLB rejects empty backend lists and falls back to last-known state if Consul unavailable

### Replication Configuration

```sql
-- Semi-sync on local DC replicas only
-- Semi-sync timeout: 500ms (reverts to async on timeout — never blocks writes indefinitely)
SET GLOBAL rpl_semi_sync_master_timeout = 500;
SET GLOBAL rpl_semi_sync_master_enabled = ON;

-- Pseudo-GTID (always-on) for flexible topology repair after failover
-- Injected by orchestrator, enables any replica to become new primary
```

### Failover Timing

```
detection + promotion + Consul update + GLB reload = 10–13s typical (up to 25s extreme)
```

**Pre-identification:** orchestrator identifies the promotion candidate *before* failure occurs. Reduces decision time during actual failover.

### pt-heartbeat (replication lag monitoring)

```bash
# Monitor replication lag
pt-heartbeat --monitor --host replica --daemonize

# Check current lag
pt-heartbeat --check --host replica

# GitHub patch: tolerate read_only state transitions + crashes without manual restart
# Critical: standard pt-heartbeat requires intervention after primary failover
```

### MySQL Slow Query Analysis

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

## Redis

### Client-Side Compression for Large Values

> Source: [DoorDash - Speeding Up Redis with Compression](https://doordash.engineering/2019/01/02/speeding-up-redis-with-compression/)

**Problem:** Large Redis values (>10KB) cause network congestion and high p99 latency at peak load. A single restaurant menu at 500KB generates significant oplog/replica traffic.

**Solution:** Compress values client-side before writing to Redis. Decompress on read. Zero Redis config change.

**Algorithm comparison for JSON payloads:**

| Algorithm | Compression ratio | Decompression speed | Use case |
|---|---|---|---|
| LZ4 | 38–40% of original | **Fastest** (2x over Snappy) | Production default |
| Snappy | 39–41% of original | Fast | Alternative |
| Zlib/Brotli | 15–25% of original | Slow | Reject for caching |

```python
# Python example: compress above threshold
import lz4.frame

COMPRESS_THRESHOLD = 10_000  # bytes

def redis_set(key, value):
    raw = json.dumps(value).encode()
    if len(raw) > COMPRESS_THRESHOLD:
        data = b'\x01' + lz4.frame.compress(raw)  # prefix byte = compressed
    else:
        data = b'\x00' + raw
    r.set(key, data)

def redis_get(key):
    data = r.get(key)
    if data[0] == 1:
        return json.loads(lz4.frame.decompress(data[1:]))
    return json.loads(data[1:])
```

```bash
# Benchmark compression algorithms on your actual data
# lzbench: https://github.com/inikep/lzbench
lzbench -t2,2 -b16 sample_payload.json

# Results format: algorithm, compression ratio, compress MB/s, decompress MB/s
```

**DoorDash results:** p99 latency dropped, Redis memory also reduced (compress-before-write benefits both network and memory simultaneously).

---

### Active Key Expiration Tuning

> Source: [Twitter/X Engineering - Improving Key Expiration in Redis](https://blog.x.com/engineering/en_us/topics/infrastructure/2019/improving-key-expiration-in-redis)

**Redis active expiration** runs via `activeExpireCycle` on a timer (several times/second). It samples N random keys with TTL per database, deletes expired ones, and repeats if >25% were expired. Default sample size: 20 keys/loop.

**Problem:** At Twitter scale (millions of keys, many databases per instance), 20 samples/loop is insufficient — expired keys accumulate, inflating memory and causing unexpected evictions.

**Symptoms:**
- Memory usage doesn't align with expected key count
- Unexpected key evictions causing cache misses
- Latency increases when Redis must evict before accepting writes
- `SCAN` over all keys temporarily fixes memory (triggers passive expiration)

**Diagnostic: check expired key overhead**
```bash
# Run a SCAN to force passive expiration, measure memory before/after
redis-cli --latency-history -i 1
redis-cli info memory | grep used_memory_human
redis-cli scan 0 count 10000 > /dev/null   # triggers passive expiration
redis-cli info memory | grep used_memory_human  # if drops significantly: expired keys accumulated

# Monitor active expiration stats
redis-cli info stats | grep expired_keys
redis-cli info stats | grep evicted_keys

# Check keyspace (how many DBs, how many keys per DB)
redis-cli info keyspace
```

**Tuning `ACTIVE_EXPIRE_CYCLE_LOOKUPS_PER_LOOP`** (Redis 3.2+):
```bash
# In redis.conf or at runtime (requires recompile for static config)
# Default: 20 — increase for dense keyspaces with many TTL keys
# Higher = more expired keys found, but higher latency

# Twitter tested: 200, 300, 500 — found 25% overhead of expired keys
# Tradeoff: 500 reduced memory 25% but pushed P99.9 latency up significantly
# 200 was a reasonable middle ground for their workload
```

**Redis version regression (2.4 → 3.2):** Version 3.2 introduced `CRON_DBS_PER_CALL` — a limit on max databases checked per expiration cycle. With many shards-as-databases, this caused expiration to miss most databases each cycle. Fix: configure or patch `CRON_DBS_PER_CALL` to match actual DB count.

**MongoDB oplog compression** (related — eBay case):

> Source: [eBay - Shopping Cart Compression](https://innovation.ebayinc.com/stories/how-ebays-shopping-cart-used-compression-techniques-to-solve-network-io-bottlenecks/)

Same principle applies to MongoDB: client-side LZ4_HIGH compression reduced oplog from **150GB/hour → 11GB/hour** (13x) and average document size from **32KB → 5KB** (6x).

```
# Rollout pattern for zero-downtime compression migration:
# Phase 1: DUAL write mode — write both compressed and uncompressed
# Phase 2: verify compressed reads on canary
# Phase 3: COMPRESS_ONLY mode — write compressed, read compressed
# Phase 4: cleanup old uncompressed fields

# Always store codec name + sizes in document metadata:
# { "codec": "LZ4_HIGH", "compressedSize": 3095, "uncompressedSize": 6485 }
# This enables seamless codec switching and forward/backward compatibility
```

## Cassandra

### Partition Design Rules

**The tombstone incident (Discord — 12 nodes, 1TB compressed/node):**
```
A Discord server deleted millions of messages, leaving 1 message in the channel.
On next read: Cassandra scanned millions of tombstones.
Result: JVM generated GC garbage faster than it could collect
→ 10-second stop-the-world GC pauses, 20-second channel load times.

Fix: track empty buckets per channel; skip them in the read path entirely.
```

**Partition key design:**
```
(channel_id, bucket)   ← time-derived integer (~10 days per bucket)
                         caps partition size under 100 MB
                         prevents GC pressure during compaction + rebalancing

Clustering key: message_id (Snowflake/UUIDv1 — time-sortable)
→ efficient descending range scans
```

**Never write null columns** — each skipped column generates a tombstone on reads. At high delete rates this compounds into GC pressure.

### Key Config Parameters

```yaml
# Reduce tombstone accumulation window (default: 10 days)
gc_grace_seconds: 172800   # 2 days — only safe if repair runs within this window

# Repair must complete within gc_grace_seconds or deleted data can resurrect
# on nodes that missed the delete (schedule nightly, off-peak)
```

### Compaction Strategy

| Strategy | When to use | Tradeoff |
|---|---|---|
| **STCS** (default) | Write-heavy, sequential | Low write amplification; poor read latency |
| **LCS** | Read-heavy, random | Predictable read latency; higher write I/O |
| **TWCS** | Time-series (TTL-based) | Minimal compaction; requires uniform TTL |

Yelp migrated ad analytics from STCS → LCS: **"vastly improved overall read performance"** by bounding the number of unmerged SSTables per read.

```yaml
compaction:
  class: 'LeveledCompactionStrategy'
  sstable_size_in_mb: 160
```

### Write Patterns

```sql
-- TTL: immutable per cell once written — plan upfront
-- Changing TTL requires full re-insertion of all data

-- UNLOGGED BATCH: safe only within same partition
-- Cross-partition batches add coordinator overhead with zero benefit
BEGIN UNLOGGED BATCH
  INSERT INTO metrics (id, ts, val) VALUES (?, ?, ?);
  INSERT INTO metrics (id, ts, val) VALUES (?, ?, ?);   -- same partition key
APPLY BATCH;
```

```bash
# Bulk historical ingest: bypass write path entirely
# sstableloader — no GC pressure, no compaction spike
sstableloader -d <seed_node> /path/to/sstables/keyspace/table/
```

**Yelp (ad analytics):** Batching writes reduced nightly batch job runtime by ~50%, network round-trips by ~10x.

### Rolling Upgrade (Spotify — 3,000 nodes)

```bash
# NEVER take down more than one node per token range simultaneously → quorum loss

# Per-node upgrade sequence (9 steps):
nodetool clearsnapshot                          # 1. free disk space
nodetool snapshot                               # 2. rollback snapshot
puppet agent --disable "upgrading cassandra"    # 3. disable config mgmt
systemctl stop cassandra                        # 4. stop
# apply upgrade                                 # 5.
systemctl start cassandra                       # 6. start
# update system.schema_columnfamilies           # 7. format migration
nodetool upgradesstables                        # 8. rewrite SSTables (HOURS per node)
# remove rollback snapshot                      # 9.

# Critical: nodetool upgradesstables can take hours
# Never leave a cluster partially upgraded (cross-version breaks repair)
```

---

## Elasticsearch

### Shard Sizing

```
Target shard size: 20–50 GB  (eBay empirical: 150 GB index → 11 shards)
Formula: shards = ceil(index_size_gb / target_shard_size_gb)

< 1 GB          → 1 shard
default/medium   → 5 shards (ES default)
> 30 GB          → increase to split
```

Scale context (eBay): 60+ clusters, 2,000+ nodes, 18B documents/day ingested, 3.5B daily search requests.

### Bulk Indexing Recipe

```json
PUT /my_index/_settings
{
  "index": {
    "refresh_interval": "60s",
    "number_of_replicas": 0,
    "translog.durability": "async",
    "translog.sync_interval": "30s"
  }
}
```

Post-load restore:
```json
{ "refresh_interval": "1s", "number_of_replicas": 1 }
```

Each replica during writes degrades throughput and increases latency linearly. Replicas=0 during bulk load, restore after indexing completes.

### Query Optimization

```
Use filter context instead of query context when scoring irrelevant → cached
Set "size": 0 on aggregation-only queries → engages shard query cache
Use stored_fields to retrieve specific fields; avoid _source on large docs
Sort by _doc instead of _score when ordering irrelevant → no score computation
Avoid wildcard queries and leading wildcards entirely
```

```bash
# Cache diagnostics
GET index_name/_stats?filter_path=indices.***.query_cache
GET index_name/_stats?filter_path=indices.***.request_cache
# High miss rate (e.g. 46GB cache, 515K hits vs 1.07M misses) → cache key misconfiguration
```

### OS Settings

```bash
sysctl -w vm.swappiness=1          # not 0 — allows OOM killer to still function
sysctl -w vm.max_map_count=262144  # ES refuses to start below this
ulimit -n 65536
echo never > /sys/kernel/mm/transparent_hugepage/enabled
echo never > /sys/kernel/mm/transparent_hugepage/defrag
```

```ini
# jvm.options: 50% of RAM, hard cap 26-30 GB (>32 GB = compressed OOPs disabled)
-Xms26g
-Xmx26g
-XX:-HeapDumpOnOutOfMemoryError    # prevent disk exhaustion on OOM
```

### TSDB Compression — Facebook Beringei (Gorilla)

**Scale:** 10B unique time series, 18M queries/minute.

- **Timestamps:** delta-of-delta encoding (exploits fixed-interval regularity)
- **Values:** XOR-based compression — XOR current vs previous, store only differing bits
- **Result:** > 90% compression ratio (lossless)

| Metric | Value |
|---|---|
| Write throughput (single machine) | 1.5M data points/sec |
| Write-to-query availability | ~300 µs |
| Read p95 latency | ~65 µs |
| Hot retention (in-memory) | 24 hours |

---

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
