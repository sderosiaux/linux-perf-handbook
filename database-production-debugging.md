# Database Production Debugging Patterns

Advanced detection queries, tuning strategies, and decision trees for LLM-driven troubleshooting of database performance issues.

**Target audience:** Production DBAs, SREs, and LLM agents performing root cause analysis.

---

## Table of Contents

1. [Hot Partition Rate Limiting](#hot-partition-rate-limiting)
2. [Prepared Statement Cache Pollution](#prepared-statement-cache-pollution)
3. [Memory-Aware Query Admission Control](#memory-aware-query-admission-control)
4. [Victim vs Culprit Query Identification](#victim-vs-culprit-query-identification)
5. [Database Time Decomposition](#database-time-decomposition)
6. [Per-Partition Metrics & Monitoring](#per-partition-metrics--monitoring)
7. [Decision Trees](#decision-trees)

---

## Hot Partition Rate Limiting

### Problem Statement

**Symptom:** Single partition receives disproportionate requests, overloading owning shards and causing cluster-wide latency degradation.

**Root cause:** Misbehaving clients, poor partition key selection, or temporal access patterns (e.g., current_date partitioning).

### Detection Queries

#### PostgreSQL - Identify Hot Partitions

```sql
-- Per-partition scan statistics
SELECT
    schemaname,
    tablename,
    seq_scan,
    seq_tup_read,
    idx_scan,
    idx_tup_fetch,
    n_tup_ins + n_tup_upd + n_tup_del as write_ops,
    round(100.0 * idx_scan / nullif(seq_scan + idx_scan, 0), 2) as idx_pct
FROM pg_stat_user_tables
WHERE tablename LIKE '%_p_%'  -- Common partition naming pattern
ORDER BY (n_tup_ins + n_tup_upd + n_tup_del) DESC
LIMIT 20;

-- Partition-level lock contention
SELECT
    c.relname as partition,
    count(DISTINCT l.pid) as waiting_sessions,
    max(now() - a.query_start) as max_wait_duration,
    array_agg(DISTINCT a.wait_event_type || ':' || a.wait_event) as wait_events
FROM pg_locks l
JOIN pg_stat_activity a ON l.pid = a.pid
JOIN pg_class c ON l.relation = c.oid
WHERE NOT l.granted
  AND c.relname LIKE '%_p_%'
GROUP BY c.relname
ORDER BY count(DISTINCT l.pid) DESC;
```

#### DynamoDB/Cassandra - Hot Key Detection (Application-Level)

```python
from collections import defaultdict
from time import time

class PartitionRateLimiter:
    """Exponential decay rate limiter for hot partition detection."""

    def __init__(self, threshold_rps=100, decay_window=60):
        self.counts = defaultdict(float)
        self.threshold = threshold_rps
        self.decay_window = decay_window
        self.last_decay = time()

    def _decay(self):
        """Apply exponential decay to all partition counts."""
        now = time()
        elapsed = now - self.last_decay
        decay_factor = 0.5 ** (elapsed / self.decay_window)

        for key in list(self.counts.keys()):
            self.counts[key] *= decay_factor
            if self.counts[key] < 0.1:
                del self.counts[key]

        self.last_decay = now

    def should_throttle(self, partition_key):
        """Returns (should_throttle: bool, current_rate: float)."""
        self._decay()
        self.counts[partition_key] += 1

        current_rate = self.counts[partition_key]
        return (current_rate > self.threshold, current_rate)

# Usage in request handler
limiter = PartitionRateLimiter(threshold_rps=100)

def handle_request(partition_key):
    should_throttle, current_rate = limiter.should_throttle(partition_key)

    if should_throttle:
        # Probabilistic rejection
        reject_prob = min(0.9, (current_rate - limiter.threshold) / limiter.threshold)
        if random.random() < reject_prob:
            raise RateLimitExceeded(
                f"Partition {partition_key} rate: {current_rate:.1f} rps "
                f"(threshold: {limiter.threshold})"
            )

    # Process request...
```

#### ScyllaDB - Native Per-Partition Rate Limiting

```sql
-- Enable per-partition rate limiting
ALTER TABLE keyspace.table WITH per_partition_rate_limit = {'max_reads_per_second': 100, 'max_writes_per_second': 100};

-- Monitor rejected requests
SELECT table_name, rejected_reads, rejected_writes
FROM system.scylladb_tables
WHERE keyspace_name = 'my_keyspace';
```

### Mitigation Strategies

| Strategy | Use Case | Implementation |
|----------|----------|----------------|
| **Probabilistic throttling** | Unpredictable hot partitions | Application-level rate limiter with exponential decay |
| **Partition key redesign** | Predictable hotspots | Add high-cardinality prefix (user_id + timestamp) |
| **Throughput redistribution** | Azure Cosmos DB | `az cosmosdb sql container redistribute-partition-throughput` |
| **Partition splitting** | Consistent hotspot exceeding limits | Split hot partition into smaller ranges |
| **Read replicas** | Read-heavy hotspots | Route reads to replicas, writes to primary |

### Tuning Parameters

```ini
# ScyllaDB
per_partition_rate_limit = {'max_reads_per_second': 100, 'max_writes_per_second': 100}

# Cassandra (via client-side throttling)
# Use TokenAwarePolicy + rate limiter per partition token range
```

**Threshold recommendations:**
- **Normal partition:** 10-50 ops/sec
- **Warning threshold:** 100 ops/sec
- **Critical threshold:** 500 ops/sec (rejection starts)

---

## Prepared Statement Cache Pollution

### Problem Statement

**Symptom:** Memory exhaustion, plan cache thrashing, generic plans executing against partitioned tables reading ALL partitions instead of one.

**Root cause:**
- One-time startup queries evicting hot production queries
- Over-caching in connection pools (250 queries × 20 connections = 5000 cached plans)
- PostgreSQL generic plans on partitioned tables

### Detection Queries

#### PostgreSQL - Plan Cache Analysis

```sql
-- Identify statements with excessive plan size (partitioned table issue)
SELECT
    substring(query, 1, 60) as query,
    calls,
    mean_exec_time,
    total_exec_time,
    pg_size_pretty(
        (SELECT sum(pg_relation_size(inhrelid))
         FROM pg_inherits
         WHERE inhparent::regclass::text = (
             SELECT tablename FROM pg_tables WHERE query LIKE '%' || tablename || '%' LIMIT 1
         ))
    ) as total_partition_size
FROM pg_stat_statements
WHERE calls > 10
  AND query LIKE '%SELECT%'
ORDER BY mean_exec_time DESC
LIMIT 20;

-- Detect generic vs custom plan usage (PostgreSQL 15+)
SELECT
    substring(query, 1, 80) as query,
    calls,
    plans,
    generic_plans,
    custom_plans,
    round(100.0 * generic_plans / nullif(plans, 0), 2) as generic_pct
FROM pg_stat_statements
WHERE plans > 0
ORDER BY generic_plans DESC
LIMIT 20;
```

#### Connection Pool Statistics

```sql
-- PgBouncer: Identify pools with excessive prepared statements
-- (Requires pgbouncer with track_extra_parameters enabled)

-- Check pool efficiency
SELECT
    database,
    user,
    cl_active + cl_waiting as client_connections,
    sv_active + sv_idle as server_connections,
    (cl_active + cl_waiting)::float / nullif(sv_active + sv_idle, 0) as pool_ratio
FROM pgbouncer.pools
WHERE pool_ratio > 20;  -- High ratio = potential cache pollution
```

#### MySQL - Prepared Statement Limits

```sql
-- Global prepared statement count
SHOW GLOBAL STATUS LIKE 'Prepared_stmt_count';
SHOW GLOBAL STATUS LIKE 'Com_stmt%';

-- Per-connection prepared statements (requires Performance Schema)
SELECT
    t.processlist_user,
    t.processlist_db,
    COUNT(ps.statement_id) as prep_stmts
FROM performance_schema.threads t
LEFT JOIN performance_schema.prepared_statements_instances ps
    ON t.thread_id = ps.owner_thread_id
WHERE t.processlist_command != 'Daemon'
GROUP BY t.processlist_user, t.processlist_db
ORDER BY prep_stmts DESC;
```

### Mitigation Strategies

#### 1. Control Generic Plan Usage (PostgreSQL 15+)

```sql
-- Force custom plans for partitioned tables
ALTER DATABASE mydb SET plan_cache_mode = 'force_custom_plan';

-- Or per-session
SET plan_cache_mode = 'force_custom_plan';

-- Options:
-- auto (default): PostgreSQL decides after 5 executions
-- force_generic_plan: Always use generic plan (predictable, may be slower)
-- force_custom_plan: Always use custom plan (parameter-sensitive)
```

#### 2. Selective Prepared Statement Usage

```python
# Python psycopg2 example
def should_prepare(query, execution_frequency):
    """Only prepare frequently-reused statements."""
    # Don't prepare:
    # - One-time queries (migrations, admin tasks)
    # - Queries with high-cardinality IN clauses
    # - Queries against partitioned tables (unless forced custom)

    if execution_frequency < 10:  # Executes less than 10 times
        return False

    if "IN (" in query and query.count(",") > 5:  # Large IN clause
        return False

    if any(partition_table in query for partition_table in PARTITIONED_TABLES):
        return False

    return True

# Usage
if should_prepare(query, get_execution_frequency(query)):
    cursor.execute(query, params)  # Prepared
else:
    cursor.execute(query % params)  # Direct execution
```

#### 3. Connection Pool Tuning

```ini
# PgBouncer - Transaction pooling to avoid per-connection cache
[pgbouncer]
pool_mode = transaction  # Not session
max_client_conn = 1000
default_pool_size = 25   # Limit server connections

# Avoid session pooling with prepared statements
```

#### 4. Statement Cache Limits

```python
# JDBC example
datasource.setMaxPoolPreparedStatementPerConnectionSize(20);  # Not 250

# Rails ActiveRecord
# Only prepare statements executed > N times
ActiveRecord::Base.connection.execute("SET plan_cache_mode = 'auto'")
```

### Decision Tree: Cache Pollution Diagnosis

```
High memory usage + slow queries on partitioned tables?
├── YES → Check generic vs custom plan ratio
│   ├── >50% generic plans on partitioned tables
│   │   └── SET plan_cache_mode = 'force_custom_plan'
│   └── <50% generic plans
│       └── Check total cached plans per connection
│           ├── >100 plans/connection
│           │   └── Reduce pool size OR use transaction pooling
│           └── <100 plans/connection
│               └── Investigate query-specific issues
└── NO → Check prepared statement count
    ├── >1000 prepared statements
    │   └── Audit which queries are prepared, apply selective preparation
    └── <1000 prepared statements
        └── Not cache pollution, check other causes
```

---

## Memory-Aware Query Admission Control

### Problem Statement

**Symptom:** Database OOM kills, query timeouts, cascading failures when concurrent queries exceed available memory.

**Root cause:** Simple concurrency limits ignore per-query memory requirements. 10 concurrent queries with 1GB each = 10GB, but 20 queries with 100MB each = 2GB.

### Detection Queries

#### PostgreSQL - Memory Usage Tracking

```sql
-- Current memory usage per query (requires pg_stat_statements)
SELECT
    pid,
    usename,
    datname,
    state,
    substring(query, 1, 60) as query,
    pg_size_pretty((query_id::text::bigint * 1024)::bigint) as est_memory,
    now() - query_start as duration
FROM pg_stat_activity
WHERE state = 'active'
ORDER BY query_start;

-- Queries historically requiring large work_mem
SELECT
    substring(query, 1, 80) as query,
    calls,
    mean_exec_time,
    -- Estimate memory from temp file usage
    pg_size_pretty(temp_blks_written * 8192) as temp_disk_used
FROM pg_stat_statements
WHERE temp_blks_written > 0
ORDER BY temp_blks_written DESC
LIMIT 20;
```

#### SQL Server 2025 - Memory Grant Tracking

```sql
-- Active queries with memory grants
SELECT
    session_id,
    request_id,
    granted_memory_kb / 1024 as granted_mb,
    query_cost,
    statement_text = SUBSTRING(
        sql.text,
        (req.statement_start_offset / 2) + 1,
        ((CASE req.statement_end_offset
            WHEN -1 THEN DATALENGTH(sql.text)
            ELSE req.statement_end_offset
        END - req.statement_start_offset) / 2) + 1
    )
FROM sys.dm_exec_requests req
CROSS APPLY sys.dm_exec_sql_text(req.sql_handle) sql
WHERE req.granted_memory_kb > 0
ORDER BY req.granted_memory_kb DESC;

-- Memory pressure history (SQL Server 2025+)
SELECT
    event_time,
    total_physical_memory_mb,
    available_physical_memory_mb,
    system_memory_state_desc
FROM sys.dm_os_memory_health_history
ORDER BY event_time DESC;
```

#### Impala - Admission Control Status

```sql
-- Check current admission queue
SHOW ADMISSION;

-- Resource pool statistics
SELECT * FROM sys.impala_query_live
WHERE admission_result != 'Admitted immediately';
```

### Implementation Patterns

#### 1. Memory-Aware Admission Controller

```python
import threading
from dataclasses import dataclass
from typing import Optional

@dataclass
class QueryMemoryEstimate:
    query_hash: str
    estimated_mb: float
    historical_max_mb: float
    confidence: float  # 0-1

class MemoryAwareAdmissionController:
    """Admits queries based on memory budget, not just concurrency."""

    def __init__(self, max_memory_mb=8192):
        self.max_memory_mb = max_memory_mb
        self.current_memory_mb = 0.0
        self.query_memory = {}  # query_id -> allocated_mb
        self.lock = threading.Lock()
        self.memory_history = {}  # query_hash -> [historical_mb]

    def estimate_query_memory(self, query: str) -> QueryMemoryEstimate:
        """Estimate memory requirement from query history."""
        query_hash = hash(query)

        if query_hash in self.memory_history:
            history = self.memory_history[query_hash]
            return QueryMemoryEstimate(
                query_hash=str(query_hash),
                estimated_mb=statistics.mean(history),
                historical_max_mb=max(history),
                confidence=min(1.0, len(history) / 10)
            )

        # Default estimate based on query type
        if "JOIN" in query.upper():
            return QueryMemoryEstimate(str(query_hash), 512, 1024, 0.3)
        elif "GROUP BY" in query.upper():
            return QueryMemoryEstimate(str(query_hash), 256, 512, 0.3)
        else:
            return QueryMemoryEstimate(str(query_hash), 64, 128, 0.3)

    def request_admission(self, query_id: str, query: str) -> tuple[bool, Optional[str]]:
        """
        Returns (admitted: bool, reason: Optional[str]).
        Blocks until memory available or timeout.
        """
        estimate = self.estimate_query_memory(query)
        required_mb = estimate.estimated_mb

        # Add safety margin for low-confidence estimates
        if estimate.confidence < 0.5:
            required_mb *= 1.5

        with self.lock:
            if self.current_memory_mb + required_mb <= self.max_memory_mb:
                self.current_memory_mb += required_mb
                self.query_memory[query_id] = required_mb
                return (True, None)
            else:
                return (False, f"Insufficient memory: need {required_mb}MB, "
                              f"available {self.max_memory_mb - self.current_memory_mb}MB")

    def release_admission(self, query_id: str, actual_memory_mb: float):
        """Release memory and update history."""
        with self.lock:
            if query_id in self.query_memory:
                allocated = self.query_memory.pop(query_id)
                self.current_memory_mb -= allocated

                # Update history for future estimates
                # (query_hash extraction omitted for brevity)

    def get_utilization(self) -> dict:
        """Current memory utilization metrics."""
        with self.lock:
            return {
                "current_mb": self.current_memory_mb,
                "max_mb": self.max_memory_mb,
                "utilization_pct": 100.0 * self.current_memory_mb / self.max_memory_mb,
                "active_queries": len(self.query_memory)
            }

# Usage
admission_controller = MemoryAwareAdmissionController(max_memory_mb=8192)

def execute_query(query_id, query):
    admitted, reason = admission_controller.request_admission(query_id, query)

    if not admitted:
        raise ResourceExhausted(reason)

    try:
        result = database.execute(query)
        actual_memory = get_query_memory_usage(query_id)
        return result
    finally:
        admission_controller.release_admission(query_id, actual_memory)
```

#### 2. Database-Native Admission Control

**PostgreSQL - Work Mem Limits:**

```sql
-- Set per-query memory limit
SET work_mem = '64MB';

-- Or per-role
ALTER ROLE reporting_user SET work_mem = '256MB';

-- Monitor queries hitting limit (spilling to disk)
SELECT
    query,
    calls,
    total_exec_time,
    temp_blks_written * 8192 / 1024 / 1024 as temp_mb
FROM pg_stat_statements
WHERE temp_blks_written > 0
ORDER BY temp_blks_written DESC;
```

**Impala Configuration:**

```sql
-- Set per-query memory limit
SET MEM_LIMIT = 8GB;

-- Pool-level admission control
CREATE RESOURCE POOL analytics_pool
    WITH (
        max_memory_resources = '50GB',
        max_queries = 10,
        queue_timeout_ms = 60000
    );
```

**CockroachDB:**

```sql
-- Enable admission control
SET CLUSTER SETTING admission.kv.enabled = true;
SET CLUSTER SETTING admission.sql_kv_response.enabled = true;

-- Monitor admission queue
SELECT * FROM crdb_internal.node_admission_control_workload;
```

### Tuning Guidelines

| Database | Parameter | Recommended Value |
|----------|-----------|-------------------|
| PostgreSQL | `work_mem` | `(RAM * 0.25) / max_connections` |
| PostgreSQL | `maintenance_work_mem` | `RAM * 0.05` (for VACUUM, etc.) |
| MySQL | `sort_buffer_size` | 256KB-2MB per connection |
| MySQL | `join_buffer_size` | 256KB-2MB per JOIN operation |
| SQL Server | Max Server Memory | 80-90% of system RAM |
| Impala | `mem_limit` | Per-query: 2-4GB; Per-node: 80% RAM |

---

## Victim vs Culprit Query Identification

### Problem Statement

**Symptom:** Single-row UPDATE taking 15 seconds. Query looks simple, profile shows no CPU usage.

**Root cause:** Query is the **victim** (waiting on locks), not the **culprit** (holding locks). Traditional profiling misleads.

### Detection Queries

#### PostgreSQL - Lock Wait Analysis

```sql
-- Identify victim and culprit queries
SELECT
    blocked.pid AS victim_pid,
    blocking.pid AS culprit_pid,
    blocked.usename AS victim_user,
    blocking.usename AS culprit_user,
    now() - blocked.query_start AS victim_wait_duration,
    now() - blocking.query_start AS culprit_duration,
    blocked.query AS victim_query,
    blocking.query AS culprit_query,
    blocked.wait_event_type || ':' || blocked.wait_event AS victim_wait_event
FROM pg_stat_activity AS blocked
JOIN pg_locks AS blocked_locks ON blocked.pid = blocked_locks.pid
JOIN pg_locks AS blocking_locks ON (
    blocking_locks.locktype = blocked_locks.locktype
    AND blocking_locks.relation = blocked_locks.relation
    AND blocking_locks.page = blocked_locks.page
    AND blocking_locks.tuple = blocked_locks.tuple
    AND blocking_locks.pid != blocked_locks.pid
)
JOIN pg_stat_activity AS blocking ON blocking_locks.pid = blocking.pid
WHERE NOT blocked_locks.granted
ORDER BY blocked.query_start;

-- Simplified version using pg_blocking_pids (PostgreSQL 9.6+)
SELECT
    blocked.pid AS victim_pid,
    blocked.query AS victim_query,
    blocking.pid AS culprit_pid,
    blocking.query AS culprit_query,
    now() - blocked.query_start AS wait_duration
FROM pg_stat_activity blocked
JOIN LATERAL unnest(pg_blocking_pids(blocked.pid)) AS blocking_pid ON true
JOIN pg_stat_activity blocking ON blocking.pid = blocking_pid
WHERE cardinality(pg_blocking_pids(blocked.pid)) > 0;
```

#### MySQL - InnoDB Lock Waits

```sql
-- Current lock waits (MySQL 8.0+)
SELECT
    waiting.trx_id AS victim_trx_id,
    waiting.trx_mysql_thread_id AS victim_thread,
    waiting.trx_query AS victim_query,
    waiting.trx_wait_started AS wait_started,
    blocking.trx_id AS culprit_trx_id,
    blocking.trx_mysql_thread_id AS culprit_thread,
    blocking.trx_query AS culprit_query,
    blocking.trx_started AS culprit_started
FROM information_schema.INNODB_TRX waiting
JOIN information_schema.INNODB_LOCK_WAITS waits
    ON waiting.trx_id = waits.requesting_trx_id
JOIN information_schema.INNODB_TRX blocking
    ON waits.blocking_trx_id = blocking.trx_id
ORDER BY waiting.trx_wait_started;

-- Performance Schema approach (more detailed)
SELECT
    object_schema,
    object_name,
    index_name,
    lock_type,
    lock_mode,
    lock_status,
    thread_id,
    processlist_id,
    processlist_info
FROM performance_schema.data_locks
WHERE lock_status = 'WAITING'
ORDER BY thread_id;
```

#### SQL Server - Deadlock and Blocking Detection

```sql
-- Current blocking chains
WITH blocking_hierarchy AS (
    SELECT
        session_id,
        blocking_session_id,
        wait_type,
        wait_time,
        wait_resource,
        0 AS level
    FROM sys.dm_exec_requests
    WHERE blocking_session_id = 0
        AND session_id IN (SELECT DISTINCT blocking_session_id FROM sys.dm_exec_requests WHERE blocking_session_id <> 0)

    UNION ALL

    SELECT
        r.session_id,
        r.blocking_session_id,
        r.wait_type,
        r.wait_time,
        r.wait_resource,
        bh.level + 1
    FROM sys.dm_exec_requests r
    INNER JOIN blocking_hierarchy bh ON r.blocking_session_id = bh.session_id
)
SELECT
    bh.session_id AS victim_session,
    bh.blocking_session_id AS culprit_session,
    bh.wait_type,
    bh.wait_time / 1000.0 AS wait_seconds,
    victim.text AS victim_query,
    culprit.text AS culprit_query
FROM blocking_hierarchy bh
CROSS APPLY sys.dm_exec_sql_text(
    (SELECT sql_handle FROM sys.dm_exec_requests WHERE session_id = bh.session_id)
) AS victim
CROSS APPLY sys.dm_exec_sql_text(
    (SELECT sql_handle FROM sys.dm_exec_requests WHERE session_id = bh.blocking_session_id)
) AS culprit
ORDER BY bh.level, bh.wait_time DESC;
```

### Response Time Attribution

#### Problem: Measuring Lock Wait Time Separately

Most databases report total response time including lock wait. You need separate metrics:

```sql
-- PostgreSQL: Distinguish execution time from lock wait
-- (Requires custom instrumentation or pg_stat_statements with wait events)

-- Enable wait event tracking
ALTER SYSTEM SET track_wait_events = on;
SELECT pg_reload_conf();

-- Query with separate timings (application-level tracking)
```

```python
# Application-level response time attribution
import time
from contextlib import contextmanager

@contextmanager
def track_response_time(query_id):
    """Track parse, execute, and lock wait time separately."""
    timings = {
        'total': 0,
        'parse': 0,
        'execute': 0,
        'lock_wait': 0
    }

    start = time.perf_counter()

    # Check if query is waiting on locks
    with connection.cursor() as cursor:
        cursor.execute("""
            SELECT wait_event_type, wait_event
            FROM pg_stat_activity
            WHERE pid = pg_backend_pid()
        """)
        wait_info = cursor.fetchone()

        if wait_info and wait_info[0] == 'Lock':
            lock_wait_start = time.perf_counter()
            yield timings
            timings['lock_wait'] = time.perf_counter() - lock_wait_start
        else:
            yield timings

    timings['total'] = time.perf_counter() - start
    timings['execute'] = timings['total'] - timings['lock_wait']

    # Log with separate dimensions
    metrics.histogram('query.time.total', timings['total'], tags=[f'query:{query_id}'])
    metrics.histogram('query.time.lock_wait', timings['lock_wait'], tags=[f'query:{query_id}'])
    metrics.histogram('query.time.execute', timings['execute'], tags=[f'query:{query_id}'])
```

### Decision Tree: Victim vs Culprit

```
Query slow despite low CPU usage?
├── YES → Check wait_event_type
│   ├── Lock:* → Query is VICTIM
│   │   ├── Identify culprit via pg_blocking_pids()
│   │   ├── Options:
│   │   │   ├── Kill culprit: SELECT pg_terminate_backend(culprit_pid)
│   │   │   ├── Reduce transaction duration in culprit
│   │   │   └── Optimize locking strategy (row-level, not table-level)
│   │   └── Monitor: response_time_source = 'lock_wait'
│   ├── IO:DataFileRead → Query is CPU-bound but IO-limited
│   │   └── Check buffer cache hit ratio, add indexes
│   └── Client:ClientRead → Query waiting for client to fetch results
│       └── Check network latency, client-side processing
└── NO → Query is CULPRIT (high CPU usage)
    └── Standard query optimization (indexes, rewrites)
```

---

## Database Time Decomposition

### Problem Statement

**Symptom:** Query takes 5 seconds end-to-end, but profiling shows 500ms of actual execution. Where did 4.5 seconds go?

**Root cause:** Database time includes parse, compile, optimize, execute, lock wait, and network transfer. Most tools only show execute time.

### Time Components

| Phase | Description | Typical % | Optimization Target |
|-------|-------------|-----------|---------------------|
| **Parse** | Syntax validation, query tree construction | 1-5% | Use prepared statements |
| **Compile/Plan** | Query optimization, plan generation | 5-20% | Enable plan caching |
| **Lock Wait** | Waiting for row/table locks | 0-80% | Reduce transaction duration, lock scope |
| **Execute** | Actual data retrieval and computation | 20-90% | Indexes, query rewrites |
| **Network** | Result set transfer to client | 1-10% | Pagination, compression |

### Detection Queries

#### PostgreSQL - Parse, Plan, Execute Breakdown

```sql
-- Enable timing statistics
SET track_activities = on;
SET track_io_timing = on;

-- View query with detailed timing (requires auto_explain)
LOAD 'auto_explain';
SET auto_explain.log_min_duration = 0;
SET auto_explain.log_analyze = on;
SET auto_explain.log_buffers = on;
SET auto_explain.log_timing = on;

-- Execute query, check logs for:
-- Planning Time: X ms
-- Execution Time: Y ms

-- Programmatic access via pg_stat_statements (aggregate)
SELECT
    substring(query, 1, 80) as query,
    calls,
    round(mean_plan_time::numeric, 2) as avg_plan_ms,
    round(mean_exec_time::numeric, 2) as avg_exec_ms,
    round((mean_plan_time / nullif(mean_plan_time + mean_exec_time, 0) * 100)::numeric, 2) as plan_pct
FROM pg_stat_statements
WHERE calls > 10
ORDER BY mean_plan_time DESC
LIMIT 20;
```

#### SQL Server - Parse and Compile Time

```sql
-- Enable statistics time
SET STATISTICS TIME ON;

-- Execute query, output shows:
-- SQL Server parse and compile time:
--   CPU time = X ms, elapsed time = Y ms.
-- SQL Server Execution Times:
--   CPU time = A ms, elapsed time = B ms.

SELECT * FROM orders WHERE customer_id = 12345;

-- Identify high compile time queries from plan cache
SELECT
    qs.sql_handle,
    qs.plan_handle,
    qs.total_worker_time / qs.execution_count as avg_cpu_us,
    qs.total_elapsed_time / qs.execution_count as avg_elapsed_us,
    -- Compile time for first execution
    qs.total_elapsed_time / qs.execution_count - qs.total_worker_time / qs.execution_count as est_compile_us,
    SUBSTRING(
        st.text,
        (qs.statement_start_offset / 2) + 1,
        ((CASE qs.statement_end_offset WHEN -1 THEN DATALENGTH(st.text)
          ELSE qs.statement_end_offset END - qs.statement_start_offset) / 2) + 1
    ) as query_text
FROM sys.dm_exec_query_stats qs
CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) st
WHERE qs.execution_count > 10
ORDER BY est_compile_us DESC;
```

#### Oracle - Parse Time Elapsed

```sql
-- Check parse time from statspack/AWR
SELECT
    sql_id,
    parse_calls,
    elapsed_time / 1000000 as total_elapsed_sec,
    cpu_time / 1000000 as total_cpu_sec,
    (elapsed_time - cpu_time) / 1000000 as wait_time_sec,
    executions
FROM v$sql
WHERE executions > 10
ORDER BY (elapsed_time - cpu_time) DESC;

-- Parse time specifically (from AWR)
SELECT
    sql_id,
    sum(parse_calls) as parse_calls,
    sum(elapsed_time_delta) / 1000000 as elapsed_sec,
    sum(parse_time_delta) / 1000000 as parse_sec,
    round(sum(parse_time_delta) / nullif(sum(elapsed_time_delta), 0) * 100, 2) as parse_pct
FROM dba_hist_sqlstat
GROUP BY sql_id
HAVING sum(parse_calls) > 100
ORDER BY parse_pct DESC;
```

### End-to-End Time Tracking

#### Application-Level Instrumentation

```python
import time
from dataclasses import dataclass
from typing import Optional

@dataclass
class QueryTiming:
    query_id: str
    parse_ms: float = 0
    plan_ms: float = 0
    execute_ms: float = 0
    lock_wait_ms: float = 0
    network_ms: float = 0
    total_ms: float = 0

    def to_dict(self):
        return {
            'query_id': self.query_id,
            'parse_ms': round(self.parse_ms, 2),
            'plan_ms': round(self.plan_ms, 2),
            'execute_ms': round(self.execute_ms, 2),
            'lock_wait_ms': round(self.lock_wait_ms, 2),
            'network_ms': round(self.network_ms, 2),
            'total_ms': round(self.total_ms, 2),
            'parse_pct': round(100 * self.parse_ms / self.total_ms, 1),
            'execute_pct': round(100 * self.execute_ms / self.total_ms, 1),
            'lock_wait_pct': round(100 * self.lock_wait_ms / self.total_ms, 1),
        }

def execute_with_timing(query: str, params: tuple = ()) -> tuple[list, QueryTiming]:
    """Execute query with detailed phase timing."""
    timing = QueryTiming(query_id=hash(query))

    start_total = time.perf_counter()

    with connection.cursor() as cursor:
        # Parse phase (prepare statement)
        start_parse = time.perf_counter()
        cursor.prepare(query)
        timing.parse_ms = (time.perf_counter() - start_parse) * 1000

        # Execute phase (includes planning + execution)
        start_exec = time.perf_counter()
        cursor.execute(None, params)  # Execute prepared
        timing.execute_ms = (time.perf_counter() - start_exec) * 1000

        # Network phase (fetch results)
        start_network = time.perf_counter()
        results = cursor.fetchall()
        timing.network_ms = (time.perf_counter() - start_network) * 1000

        # Check for lock wait (PostgreSQL-specific)
        cursor.execute("""
            SELECT COALESCE(
                EXTRACT(EPOCH FROM (now() - state_change)) * 1000, 0
            )
            FROM pg_stat_activity
            WHERE pid = pg_backend_pid()
              AND wait_event_type = 'Lock'
        """)
        timing.lock_wait_ms = cursor.fetchone()[0] or 0

    timing.total_ms = (time.perf_counter() - start_total) * 1000

    # Log breakdown
    logger.info("Query timing breakdown", extra=timing.to_dict())

    return results, timing

# Usage
results, timing = execute_with_timing("SELECT * FROM orders WHERE id = %s", (12345,))

if timing.lock_wait_ms > timing.execute_ms:
    logger.warning(f"Query {timing.query_id} spent {timing.lock_wait_pct}% waiting on locks")
```

### Visualization: Time Waterfall

```python
import matplotlib.pyplot as plt
import numpy as np

def plot_query_waterfall(timings: list[QueryTiming]):
    """Visualize query time breakdown as stacked bar chart."""
    queries = [t.query_id for t in timings]
    parse = [t.parse_ms for t in timings]
    plan = [t.plan_ms for t in timings]
    execute = [t.execute_ms for t in timings]
    lock_wait = [t.lock_wait_ms for t in timings]
    network = [t.network_ms for t in timings]

    fig, ax = plt.subplots(figsize=(12, 6))

    x = np.arange(len(queries))
    width = 0.8

    p1 = ax.bar(x, parse, width, label='Parse')
    p2 = ax.bar(x, plan, width, bottom=parse, label='Plan')
    p3 = ax.bar(x, execute, width, bottom=np.array(parse)+np.array(plan), label='Execute')
    p4 = ax.bar(x, lock_wait, width,
                bottom=np.array(parse)+np.array(plan)+np.array(execute), label='Lock Wait')
    p5 = ax.bar(x, network, width,
                bottom=np.array(parse)+np.array(plan)+np.array(execute)+np.array(lock_wait),
                label='Network')

    ax.set_ylabel('Time (ms)')
    ax.set_title('Query Time Breakdown')
    ax.set_xticks(x)
    ax.set_xticklabels([f'Q{i+1}' for i in range(len(queries))], rotation=45)
    ax.legend()

    plt.tight_layout()
    plt.savefig('query_waterfall.png')
```

### Optimization Decision Matrix

| Dominant Phase | % of Total | Action |
|----------------|------------|--------|
| **Parse** | >10% | Use prepared statements, connection pooling |
| **Plan** | >20% | Enable plan caching (`plan_cache_mode = auto`) |
| **Lock Wait** | >30% | Reduce transaction scope, use row-level locks, optimize culprit queries |
| **Execute** | >60% | Standard optimization: indexes, query rewrites, partition pruning |
| **Network** | >15% | Use pagination, enable compression, reduce column count |

---

## Per-Partition Metrics & Monitoring

### Problem Statement

**Symptom:** Aggregate metrics show healthy database (50% CPU, 70% disk util), but user reports timeouts.

**Root cause:** Hotspot hidden by aggregation. One partition at 100% while others idle. Disaggregation reveals the truth.

### Monitoring Strategy

#### 1. Partition-Level Metrics (PostgreSQL)

```sql
-- Create monitoring view for partitioned tables
CREATE VIEW partition_metrics AS
SELECT
    schemaname,
    tablename,
    -- Read metrics
    seq_scan,
    seq_tup_read,
    idx_scan,
    idx_tup_fetch,
    -- Write metrics
    n_tup_ins as inserts,
    n_tup_upd as updates,
    n_tup_del as deletes,
    n_tup_hot_upd as hot_updates,
    -- Bloat metrics
    n_live_tup,
    n_dead_tup,
    round(100.0 * n_dead_tup / nullif(n_live_tup + n_dead_tup, 0), 2) as dead_pct,
    -- Vacuum metrics
    last_vacuum,
    last_autovacuum,
    -- Size
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) as total_size
FROM pg_stat_user_tables
WHERE tablename LIKE '%_p_%'  -- Partition naming convention
ORDER BY (n_tup_ins + n_tup_upd + n_tup_del) DESC;

-- Query partition metrics
SELECT * FROM partition_metrics WHERE inserts + updates + deletes > 1000;
```

#### 2. Time-Series Partition Metrics (Prometheus + PostgreSQL Exporter)

```yaml
# prometheus.yml - scrape per-partition metrics
scrape_configs:
  - job_name: 'postgresql_partitions'
    static_configs:
      - targets: ['localhost:9187']
    metrics_path: /metrics
    relabel_configs:
      - source_labels: [table]
        regex: '(.+)_p_(.+)'
        target_label: partition
        replacement: '$2'
```

```sql
-- Custom exporter query (postgres_exporter)
-- File: queries.yaml
pg_partition_operations:
  query: |
    SELECT
      schemaname,
      tablename,
      n_tup_ins + n_tup_upd + n_tup_del as write_ops,
      seq_scan + idx_scan as read_ops,
      n_dead_tup
    FROM pg_stat_user_tables
    WHERE tablename LIKE '%_p_%'
  metrics:
    - schemaname:
        usage: "LABEL"
        description: "Schema name"
    - tablename:
        usage: "LABEL"
        description: "Partition name"
    - write_ops:
        usage: "COUNTER"
        description: "Total write operations"
    - read_ops:
        usage: "COUNTER"
        description: "Total read operations"
    - n_dead_tup:
        usage: "GAUGE"
        description: "Number of dead tuples"
```

#### 3. DynamoDB CloudWatch Metrics (Per-Partition)

```python
import boto3
from datetime import datetime, timedelta

def get_partition_metrics(table_name, start_time, end_time):
    """Fetch per-partition consumed capacity from CloudWatch."""
    cloudwatch = boto3.client('cloudwatch')

    # Partition-level read capacity
    response = cloudwatch.get_metric_statistics(
        Namespace='AWS/DynamoDB',
        MetricName='ConsumedReadCapacityUnits',
        Dimensions=[
            {'Name': 'TableName', 'Value': table_name}
        ],
        StartTime=start_time,
        EndTime=end_time,
        Period=60,  # 1-minute granularity
        Statistics=['Sum', 'Maximum', 'Average']
    )

    # Identify hot partitions (top 10% consumption)
    datapoints = sorted(response['Datapoints'],
                       key=lambda x: x['Maximum'],
                       reverse=True)

    hot_threshold = datapoints[0]['Maximum'] * 0.5
    hot_partitions = [dp for dp in datapoints if dp['Maximum'] > hot_threshold]

    return hot_partitions

# Usage
hot = get_partition_metrics(
    'orders_table',
    datetime.now() - timedelta(hours=1),
    datetime.now()
)

for partition in hot:
    print(f"Hot partition detected: {partition['Timestamp']} "
          f"consumed {partition['Maximum']} RCU")
```

#### 4. Cassandra/ScyllaDB Per-Partition Statistics

```cql
-- Enable per-partition monitoring in ScyllaDB
ALTER TABLE keyspace.table WITH compaction = {
    'class': 'SizeTieredCompactionStrategy',
    'enabled': true,
    'log_all': true
};

-- Query partition statistics
SELECT * FROM system.large_partitions WHERE keyspace_name = 'my_keyspace';

-- Monitor partition size distribution
SELECT partition_key, rows, size_bytes
FROM system.size_estimates
WHERE keyspace_name = 'my_keyspace'
  AND table_name = 'orders'
ORDER BY size_bytes DESC
LIMIT 20;
```

### Alerting Rules

#### Prometheus AlertManager Rules

```yaml
groups:
  - name: partition_alerts
    interval: 60s
    rules:
      # Hot partition detection (write rate)
      - alert: PartitionWriteHotspot
        expr: |
          rate(pg_partition_write_ops[5m]) > 100
          AND
          rate(pg_partition_write_ops[5m]) /
          avg(rate(pg_partition_write_ops[5m])) > 5
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Hot partition detected: {{ $labels.tablename }}"
          description: "Partition {{ $labels.tablename }} receiving {{ $value }} writes/sec (5x average)"

      # Partition bloat
      - alert: PartitionBloat
        expr: pg_partition_dead_tup_pct > 20
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Partition bloat: {{ $labels.tablename }}"
          description: "{{ $labels.tablename }} has {{ $value }}% dead tuples"

      # Partition size imbalance
      - alert: PartitionSkew
        expr: |
          max(pg_partition_size_bytes) / avg(pg_partition_size_bytes) > 10
        for: 15m
        labels:
          severity: info
        annotations:
          summary: "Partition size skew detected"
          description: "Largest partition is {{ $value }}x average size"
```

### Grafana Dashboard Template

```json
{
  "dashboard": {
    "title": "Per-Partition Database Metrics",
    "panels": [
      {
        "title": "Top 10 Partitions by Write Rate",
        "targets": [
          {
            "expr": "topk(10, rate(pg_partition_write_ops[5m]))",
            "legendFormat": "{{ tablename }}"
          }
        ],
        "type": "graph"
      },
      {
        "title": "Partition Size Distribution",
        "targets": [
          {
            "expr": "pg_partition_size_bytes",
            "legendFormat": "{{ tablename }}"
          }
        ],
        "type": "heatmap"
      },
      {
        "title": "Partition Skew Ratio",
        "targets": [
          {
            "expr": "max(pg_partition_size_bytes) / avg(pg_partition_size_bytes)",
            "legendFormat": "Skew Ratio"
          }
        ],
        "type": "stat",
        "thresholds": [
          {"value": 1, "color": "green"},
          {"value": 5, "color": "yellow"},
          {"value": 10, "color": "red"}
        ]
      }
    ]
  }
}
```

---

## Decision Trees

### 1. Slow Query Root Cause Analysis

```
Query slow (response time > threshold)?
├── Check CPU usage in profile
│   ├── High CPU (>80% of response time)
│   │   ├── Query is CULPRIT (compute-bound)
│   │   │   ├── Check EXPLAIN plan
│   │   │   │   ├── Seq Scan? → Add index
│   │   │   │   ├── Large sort? → Increase work_mem or add index
│   │   │   │   └── Hash join? → Check join selectivity, update stats
│   │   │   └── Profile shows expensive function?
│   │   │       └── Optimize function or push to application
│   │   └── Low CPU (<20% of response time)
│   │       ├── Query is VICTIM (waiting)
│   │       │   ├── Check wait_event_type
│   │       │   │   ├── Lock → Identify culprit with pg_blocking_pids
│   │       │   │   ├── IO → Check cache hit ratio, add indexes
│   │       │   │   └── Client → Network latency or slow client processing
│   │       │   └── Decompose time: parse/plan/execute/lock/network
│   │       └── Mid CPU (20-80%)
│   │           └── Mixed workload, check all phases
└── Check aggregate vs per-partition metrics
    ├── Aggregate looks healthy (50% CPU)
    │   └── Disaggregate: Is one partition at 100%?
    │       ├── YES → Hot partition problem
    │       │   └── Apply rate limiting, partition key redesign
    │       └── NO → Check temporal patterns (oscillation)
    └── Aggregate shows saturation
        └── Scale up or optimize top queries
```

### 2. Memory Pressure Diagnosis

```
Database OOM or query timeouts?
├── Check total allocated memory
│   ├── sum(work_mem * connections) > available_memory?
│   │   ├── YES → Reduce work_mem or max_connections
│   │   │   └── OR implement memory-aware admission control
│   │   └── NO → Check individual query memory
│   │       ├── Queries with temp_blks_written > 0?
│   │       │   ├── YES → Increase work_mem for these specific queries
│   │       │   │   └── OR add indexes to avoid sorts/hashes
│   │       │   └── NO → Check buffer pool size
│   │       └── Buffer pool hit ratio < 95%?
│   │           ├── YES → Increase shared_buffers (PostgreSQL)
│   │           │        or innodb_buffer_pool_size (MySQL)
│   │           └── NO → Memory leak in application?
└── Check prepared statement cache
    ├── >1000 prepared statements cached?
    │   ├── YES → Cache pollution
    │   │   ├── Check plan_cache_mode (force custom for partitioned)
    │   │   ├── Reduce connection pool size
    │   │   └── Audit which queries are prepared
    │   └── NO → Check connection pooling
    │       └── Connection count * statements_per_connection < 10,000?
```

### 3. Lock Contention Resolution

```
Transactions timing out or deadlocking?
├── Identify pattern
│   ├── Frequent deadlocks (>1/minute)?
│   │   ├── Check deadlock logs for cycle
│   │   ├── Consistent pattern?
│   │   │   ├── YES → Application bug (inconsistent lock order)
│   │   │   │   └── Fix: Always lock tables in same order
│   │   │   └── NO → High concurrency on shared resources
│   │   │       └── Reduce transaction scope, use optimistic locking
│   │   └── Enable log_lock_waits for details
│   └── Long lock waits (>5 seconds)?
│       ├── Identify victim vs culprit
│       │   ├── Use pg_blocking_pids() or INNODB_LOCK_WAITS
│       │   ├── Culprit is long-running transaction?
│       │   │   ├── YES → Reduce transaction duration
│       │   │   │   ├── Break into smaller transactions
│       │   │   │   └── Move non-DB work outside transaction
│       │   │   └── NO → Culprit holding unnecessary locks?
│       │   │       └── Use row-level locking (FOR UPDATE) not table locks
│       │   └── Victim queries cumulative?
│       │       └── Implement statement timeout: SET statement_timeout = '30s'
│       └── Monitor response_time_source metric
│           └── High 'lock_wait' percentage → Focus on culprit optimization
```

### 4. Prepared Statement Cache Pollution

```
High memory usage + inconsistent query performance?
├── Check prepared statement count
│   ├── >1000 statements cached?
│   │   ├── YES → Cache pollution confirmed
│   │   │   ├── Check queries on partitioned tables
│   │   │   │   ├── Generic plans reading all partitions?
│   │   │   │   │   └── SET plan_cache_mode = 'force_custom_plan'
│   │   │   │   └── Queries with plan_time > exec_time?
│   │   │   │       └── Disable caching for these (direct execution)
│   │   │   ├── Check connection pool
│   │   │   │   └── connections * statements_per_conn > 5000?
│   │   │   │       ├── Reduce pool size
│   │   │   │       └── OR use transaction pooling (PgBouncer)
│   │   │   └── Audit prepared statements
│   │   │       └── Only prepare high-frequency queries (>10 executions)
│   │   └── NO → Check other memory consumers
│   │       └── Buffer pool, sort buffers, connection memory
└── Monitor plan cache metrics
    ├── PostgreSQL: generic_plans / total_plans ratio
    ├── MySQL: Prepared_stmt_count
    └── SQL Server: sys.dm_exec_cached_plans size
```

### 5. Hot Partition Mitigation

```
One partition receiving disproportionate traffic?
├── Characterize hotspot
│   ├── Predictable (e.g., current_date partition)?
│   │   ├── YES → Static mitigation
│   │   │   ├── Azure Cosmos DB: Redistribute throughput to hot partition
│   │   │   ├── DynamoDB: Use partition key with high cardinality prefix
│   │   │   └── Cassandra: Change partition key (bucketing strategy)
│   │   └── NO → Unpredictable hotspot
│   │       └── Dynamic mitigation
│   │           ├── Application-level rate limiting with exponential decay
│   │           ├── ScyllaDB: ALTER TABLE with per_partition_rate_limit
│   │           └── Monitor: Track per-partition ops/sec with decay
│   ├── Read-heavy or write-heavy?
│   │   ├── Read-heavy → Add read replicas
│   │   └── Write-heavy → Partition splitting or redesign key
│   └── Temporary spike or sustained?
│       ├── Temporary → Throttle requests, backpressure
│       └── Sustained → Architectural fix (key redesign)
└── Implement monitoring
    └── Alert on: partition_rate > 5x avg_partition_rate
```

---

## Production Troubleshooting Playbook

### Scenario 1: Query Suddenly Slow (No Code Changes)

**Symptoms:** P95 latency increased from 50ms to 5000ms overnight.

**Investigation Steps:**

1. **Check statistics freshness:**
   ```sql
   -- PostgreSQL
   SELECT schemaname, tablename, last_analyze, last_autoanalyze
   FROM pg_stat_user_tables
   WHERE last_analyze < now() - interval '7 days';

   -- Fix: ANALYZE tablename;
   ```

2. **Check plan changes:**
   ```sql
   -- Compare current plan with historical
   EXPLAIN ANALYZE <query>;
   -- Look for sudden switch from Index Scan to Seq Scan
   ```

3. **Check lock contention:**
   ```sql
   -- Sudden spike in lock waits?
   SELECT count(*) FROM pg_locks WHERE NOT granted;
   ```

4. **Check autovacuum:**
   ```sql
   -- Table bloat from dead tuples?
   SELECT n_dead_tup, n_live_tup,
          round(100.0 * n_dead_tup / nullif(n_live_tup + n_dead_tup, 0), 2) as dead_pct
   FROM pg_stat_user_tables
   WHERE tablename = 'target_table';
   ```

### Scenario 2: Intermittent Timeouts (80% success, 20% timeout)

**Symptoms:** Same query succeeds quickly most of the time, occasionally times out.

**Investigation Steps:**

1. **Check for lock waits on specific rows:**
   ```sql
   -- Victim queries
   SELECT blocked.query, blocking.query,
          now() - blocked.query_start as wait_duration
   FROM pg_stat_activity blocked
   JOIN LATERAL unnest(pg_blocking_pids(blocked.pid)) blocking_pid ON true
   JOIN pg_stat_activity blocking ON blocking.pid = blocking_pid;
   ```

2. **Check for periodic autovacuum:**
   ```sql
   -- Autovacuum runs coincide with timeouts?
   SELECT * FROM pg_stat_progress_vacuum;
   ```

3. **Check for batch operations:**
   ```sql
   -- Large batch INSERT/UPDATE at regular intervals?
   SELECT query, query_start, state
   FROM pg_stat_activity
   WHERE state = 'active'
     AND query NOT LIKE '%pg_stat_activity%'
   ORDER BY query_start;
   ```

### Scenario 3: High Memory Usage Leading to OOM

**Symptoms:** Database process killed by OOM killer, workload unchanged.

**Investigation Steps:**

1. **Check work_mem * connections:**
   ```sql
   -- Total possible memory allocation
   SELECT
       current_setting('max_connections')::int *
       pg_size_bytes(current_setting('work_mem')) / 1024 / 1024 / 1024 as max_work_mem_gb;
   ```

2. **Check prepared statement cache:**
   ```sql
   -- Count cached plans
   SELECT count(*) FROM pg_prepared_statements;
   ```

3. **Check queries spilling to disk:**
   ```sql
   SELECT query, temp_blks_written, calls
   FROM pg_stat_statements
   WHERE temp_blks_written > 0
   ORDER BY temp_blks_written DESC;
   ```

---

## References & Further Reading

### Hot Partition Rate Limiting
- [ScyllaDB: Per-Partition Query Rate Limiting](https://www.scylladb.com/2024/04/17/per-partition-query-rate-limiting/)
- [Azure Cosmos DB: Redistribute Throughput Across Partitions](https://learn.microsoft.com/en-us/azure/cosmos-db/nosql/how-to-redistribute-throughput-across-partitions)
- [DynamoDB Hot Partition Definition](https://www.scylladb.com/glossary/dynamodb-hot-partition/)

### Prepared Statement Cache
- [Deep Dive: PostgreSQL Prepared Statements Memory Exhaustion](https://andrewbaker.ninja/2025/11/21/deep-dive-into-postgresql-prepared-statements-when-plan-caching-goes-wrong-leading-to-memory-exhaustion/)
- [PostgreSQL JDBC Statement Caching - Vlad Mihalcea](https://vladmihalcea.com/postgresql-jdbc-statement-caching/)
- [YugabyteDB: plan_cache_mode](https://yugabytedb.tips/taming-prepared-statement-re%E2%80%91planning-with-plan_cache_mode-in-yugabytedb/)

### Memory-Aware Admission Control
- [Resource-Adaptive Query Execution (CIDR 2025)](https://www.vldb.org/cidrdb/papers/2025/p2-otaki.pdf)
- [Impala Admission Control](https://impala.apache.org/docs/build/html/topics/impala_admission.html)
- [CockroachDB Admission Control](https://www.cockroachlabs.com/docs/stable/admission-control)
- [SQL Server 2025 Memory Troubleshooting](https://www.brentozar.com/archive/2025/09/sql-server-2025-makes-memory-troubleshooting-easier/)

### Lock Wait Analysis
- [SQL Server Deadlocks Guide](https://learn.microsoft.com/en-us/sql/relational-databases/sql-server-deadlocks-guide?view=sql-server-ver17)
- [Understanding Wait Events in PostgreSQL](https://stormatics.tech/blogs/understanding-wait-events-in-postgresql)

### Database Time Decomposition
- [SQL Server: Compilation vs Execution Time](https://techcommunity.microsoft.com/t5/azure-database-support-blog/lesson-learned-286-compilation-vs-execution-time-running-a-tsql/ba-p/3716502)
- [SET STATISTICS TIME ON - SQL Server](https://www.sqlshack.com/usage-details-of-the-set-statistics-time-on-statement-in-sql-server/)

### Per-Partition Monitoring
- [Database Observability: Monitoring Queries (2025)](https://dasroot.net/posts/2025/12/database-observability-monitoring/)
- [Database Monitoring Metrics Guide (2025)](https://last9.io/blog/database-monitoring-metrics/)
- [Mastering pg_stat_activity for PostgreSQL](https://www.instaclustr.com/blog/mastering-pg-stat-activity-for-real-time-monitoring-in-postgresql/)

---

## Glossary

| Term | Definition |
|------|------------|
| **Hot partition** | A partition receiving disproportionate traffic compared to others, causing localized overload |
| **Victim query** | A query waiting on resources (locks, I/O) held by another query (the culprit) |
| **Culprit query** | A query holding resources (locks, connections) that block other queries |
| **Prepared statement cache** | In-memory storage of compiled query plans to avoid re-parsing and re-planning |
| **Generic plan** | A cached query plan that works for any parameter values (may be suboptimal for partitioned tables) |
| **Custom plan** | A query plan optimized for specific parameter values (recomputed each execution) |
| **Work_mem** | PostgreSQL memory allocated per query operation (sort, hash, etc.) |
| **Admission control** | Gating mechanism to limit concurrent queries based on resource availability |
| **Lock wait time** | Duration a transaction spends waiting for locks held by other transactions |
| **Parse time** | Time spent validating query syntax and building parse tree |
| **Plan time** | Time spent by query optimizer generating execution plan |
| **Execute time** | Time spent actually retrieving and processing data |
| **Partition skew** | Imbalance in data distribution or access patterns across partitions |
| **Response time attribution** | Decomposing total latency into constituent phases (parse, plan, lock wait, execute, network) |

---

**Version:** 1.0
**Last Updated:** 2026-02-03
**Maintainer:** For updates, see `/Users/sderosiaux/code/ai-projects/linux-perf-handbook/`
