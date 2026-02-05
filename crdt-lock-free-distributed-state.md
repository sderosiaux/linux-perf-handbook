# CRDTs: Lock-Free Distributed State

Conflict-free Replicated Data Types for eventually consistent, coordination-free distributed systems.

**Target audience:** Engineers building distributed systems where lock contention is a bottleneck.

---

## Table of Contents

1. [When CRDTs vs Locks](#when-crdts-vs-locks)
2. [Core CRDT Types](#core-crdt-types)
3. [State vs Delta Representation](#state-vs-delta-representation)
4. [Storage Schema Patterns](#storage-schema-patterns)
5. [Implementation Examples](#implementation-examples)
6. [Decision Trees](#decision-trees)
7. [Performance Characteristics](#performance-characteristics)

---

## When CRDTs vs Locks

### The Lock Problem at Scale

Lock contention degrades non-linearly. At high concurrency:

```
Lock Contention Impact:
Threads  | Lock Wait (relative)
---------|---------------------
    1    | 1x
   10    | 8x
  100    | 50x
 1000    | 200x+
```

**Symptoms indicating CRDT consideration:**
- `futex` wait dominating off-CPU time
- Cross-datacenter writes with >100ms RTT
- Frequent deadlocks in distributed transactions
- Write throughput limited by coordination

### Trade-off Matrix

| Aspect | Locks | CRDTs |
|--------|-------|-------|
| Consistency | Strong (linearizable) | Eventual (convergent) |
| Latency | Bounded by slowest node | Local writes, async sync |
| Availability | Blocked during partition | Always available |
| Complexity | Simple semantics | Type-specific semantics |
| Use case | Banking, inventory | Counters, collaboration, caches |

### Decision Criteria

```
Need CRDTs when:
├── Multi-datacenter writes required
├── Network partitions must not block writes
├── P99 latency cannot tolerate coordination
├── Conflict resolution is semantically meaningful
└── Read-your-writes consistency per replica is sufficient

Need Locks when:
├── Invariants must never be violated (bank balance >= 0)
├── Ordering between operations is critical
├── Single-datacenter or low-latency network
└── Strong consistency required for correctness
```

---

## Core CRDT Types

### G-Counter (Grow-only Counter)

Monotonically increasing counter. Each node maintains its own partial count.

```
State: Map<node_id, count>

Example (3 nodes):
  Node A: {A: 5, B: 0, C: 0}
  Node B: {A: 3, B: 7, C: 0}
  Node C: {A: 3, B: 2, C: 4}

After merge:
  {A: 5, B: 7, C: 4} → Value = 16
```

**Merge function:** `max(local[node], remote[node])` per node

**Invariant:** Only the owning node increments its entry.

**Use cases:**
- Page view counters
- Event counts
- Metrics aggregation

### PN-Counter (Positive-Negative Counter)

Supports increment AND decrement via two G-Counters.

```
State: {
  P: Map<node_id, count>,  // increments
  N: Map<node_id, count>   // decrements
}

Value = sum(P) - sum(N)

Example:
  P: {A: 10, B: 5}  → +15
  N: {A: 3, B: 2}   → -5
  Value: 10
```

**Merge function:** Merge P and N independently as G-Counters.

**Use cases:**
- Like/dislike counts
- Stock levels (with caveats)
- Active user counts

**Caveat:** Cannot enforce `value >= 0`. Use bounded counters or application-level checks.

### LWW-Register (Last-Writer-Wins Register)

Single value with timestamp-based conflict resolution.

```
State: {
  value: T,
  timestamp: LogicalClock,
  node_id: string  // tie-breaker
}

Merge: if (remote.timestamp > local.timestamp) OR
          (remote.timestamp == local.timestamp AND remote.node_id > local.node_id)
       then use remote
```

**Clock types:**
- **Lamport timestamp:** Simple, no wall-clock dependency
- **HLC (Hybrid Logical Clock):** Lamport + physical time, better debugging
- **Physical timestamp:** Requires synchronized clocks (NTP)

**Use cases:**
- User profile fields
- Configuration values
- Session data

**Anti-pattern:** Using LWW for collections (use OR-Set instead).

### OR-Set (Observed-Remove Set)

Add and remove elements with causal tracking. Add-wins semantics on concurrent add/remove.

```
State (Dot-based, modern):
{
  entries: Map<Dot, Element>,  // Dot = (node_id, clock)
  context: VectorClock         // compact tombstone tracking
}

Operations:
  add(e):    entries[(node_id, ++clock)] = e
  remove(e): delete all entries where value == e
             merge into context
  lookup(e): exists in entries

Example:
  Node A adds "x": entries = {(A,1): "x"}
  Node B adds "x": entries = {(B,1): "x"}
  Node A removes "x": entries = {}, context = {A:1}

  After merge: entries = {(B,1): "x"} (B's add survives A's remove)
```

**Merge function:**
1. Union of entries not dominated by either context
2. Join of contexts

**Use cases:**
- Shopping carts
- Collaborative editing (character sets)
- Tagging systems
- Membership lists

### MV-Register (Multi-Value Register)

Preserves all concurrent writes instead of picking one.

```
State: Set<{value: T, vector_clock: VectorClock}>

Merge: Keep all values where neither dominates the other

Example:
  Node A: {value: "foo", vc: {A:1}}
  Node B: {value: "bar", vc: {B:1}}

  Merged: [{value: "foo", vc: {A:1}}, {value: "bar", vc: {B:1}}]

  Application must resolve: show both, let user pick, or merge semantically
```

**Use cases:**
- Document fields where conflicts need user resolution
- Version vectors
- Riak bucket values

---

## State vs Delta Representation

### State-Based (CvRDT)

Send full state on every sync. Simple but bandwidth-heavy.

```
Sync Protocol:
  1. Node A sends entire state to Node B
  2. Node B merges: state_B = merge(state_B, state_A)
  3. Repeat bidirectionally
```

**Pros:** Idempotent, tolerates message loss/reorder, simple implementation
**Cons:** O(state_size) per sync, wasteful for large states

### Delta-State (δ-CRDT)

Send only mutations since last sync. Best of both worlds.

```
Delta Protocol:
  1. Node A tracks delta since last sync to B
  2. Node A sends delta (minimal state change)
  3. Node B merges: state_B = merge(state_B, delta_A)
  4. Delta is itself a valid CRDT state

Example (G-Counter):
  Full state: {A: 100, B: 50, C: 75}
  Delta after A increments: {A: 1}  // just the change
```

**Delta requirements:**
- Delta must be a valid state (join semi-lattice element)
- `merge(state, delta)` = same as `merge(state, full_remote_state)`
- Deltas can be composed: `delta_1_to_3 = merge(delta_1_to_2, delta_2_to_3)`

### Comparison

| Aspect | State-Based | Delta-State |
|--------|-------------|-------------|
| Bandwidth | O(state_size) | O(delta_size) |
| Complexity | Simple | Moderate |
| Failure recovery | Send full state | May need full state |
| Memory | State only | State + delta buffer |

**Recommendation:** Use delta-state for production. Fall back to full state sync on reconnect or buffer overflow.

---

## Storage Schema Patterns

### Generic CRDT Table

```sql
CREATE TABLE crdt_state (
    entity_id    UUID NOT NULL,       -- logical entity identifier
    crdt_type    VARCHAR(32) NOT NULL, -- 'g_counter', 'lww_register', etc.
    node_id      VARCHAR(64) NOT NULL, -- replica/actor identifier
    clock        BIGINT NOT NULL,      -- logical clock (Lamport/HLC)
    payload      JSONB NOT NULL,       -- type-specific data
    updated_at   TIMESTAMPTZ DEFAULT now(),

    PRIMARY KEY (entity_id, node_id, clock)
);

CREATE INDEX idx_crdt_entity ON crdt_state(entity_id);
CREATE INDEX idx_crdt_updated ON crdt_state(updated_at);
```

### G-Counter Schema

```sql
CREATE TABLE g_counter (
    counter_id   VARCHAR(255) NOT NULL,
    node_id      VARCHAR(64) NOT NULL,
    count        BIGINT NOT NULL DEFAULT 0,
    updated_at   TIMESTAMPTZ DEFAULT now(),

    PRIMARY KEY (counter_id, node_id)
);

-- Read counter value
SELECT counter_id, SUM(count) as value
FROM g_counter
WHERE counter_id = $1
GROUP BY counter_id;

-- Increment (only own node)
INSERT INTO g_counter (counter_id, node_id, count)
VALUES ($1, $2, 1)
ON CONFLICT (counter_id, node_id)
DO UPDATE SET count = g_counter.count + 1, updated_at = now();

-- Merge from remote
INSERT INTO g_counter (counter_id, node_id, count, updated_at)
VALUES ($1, $2, $3, $4)
ON CONFLICT (counter_id, node_id)
DO UPDATE SET
    count = GREATEST(g_counter.count, EXCLUDED.count),
    updated_at = GREATEST(g_counter.updated_at, EXCLUDED.updated_at);
```

### LWW-Register Schema

```sql
CREATE TABLE lww_register (
    register_id  VARCHAR(255) NOT NULL PRIMARY KEY,
    value        JSONB,
    clock        BIGINT NOT NULL,          -- Lamport timestamp
    node_id      VARCHAR(64) NOT NULL,     -- tie-breaker
    updated_at   TIMESTAMPTZ DEFAULT now()
);

-- Write (compare-and-swap semantics)
INSERT INTO lww_register (register_id, value, clock, node_id)
VALUES ($1, $2, $3, $4)
ON CONFLICT (register_id)
DO UPDATE SET
    value = CASE
        WHEN EXCLUDED.clock > lww_register.clock
             OR (EXCLUDED.clock = lww_register.clock AND EXCLUDED.node_id > lww_register.node_id)
        THEN EXCLUDED.value
        ELSE lww_register.value
    END,
    clock = GREATEST(lww_register.clock, EXCLUDED.clock),
    node_id = CASE
        WHEN EXCLUDED.clock > lww_register.clock
             OR (EXCLUDED.clock = lww_register.clock AND EXCLUDED.node_id > lww_register.node_id)
        THEN EXCLUDED.node_id
        ELSE lww_register.node_id
    END,
    updated_at = now();
```

### OR-Set Schema (Dot-based)

```sql
CREATE TABLE or_set (
    set_id       VARCHAR(255) NOT NULL,
    element      JSONB NOT NULL,
    dot_node     VARCHAR(64) NOT NULL,   -- actor that added
    dot_clock    BIGINT NOT NULL,        -- clock when added

    PRIMARY KEY (set_id, dot_node, dot_clock)
);

CREATE TABLE or_set_context (
    set_id       VARCHAR(255) NOT NULL,
    node_id      VARCHAR(64) NOT NULL,
    max_clock    BIGINT NOT NULL DEFAULT 0,

    PRIMARY KEY (set_id, node_id)
);

-- Add element
WITH new_clock AS (
    INSERT INTO or_set_context (set_id, node_id, max_clock)
    VALUES ($1, $2, 1)
    ON CONFLICT (set_id, node_id)
    DO UPDATE SET max_clock = or_set_context.max_clock + 1
    RETURNING max_clock
)
INSERT INTO or_set (set_id, element, dot_node, dot_clock)
SELECT $1, $3, $2, max_clock FROM new_clock;

-- Remove element (delete all dots for this element)
DELETE FROM or_set
WHERE set_id = $1 AND element = $2;

-- Lookup element
SELECT EXISTS(
    SELECT 1 FROM or_set WHERE set_id = $1 AND element = $2
);

-- Get all elements
SELECT DISTINCT element FROM or_set WHERE set_id = $1;
```

### Delta Sync Table

```sql
CREATE TABLE crdt_delta_log (
    delta_id     BIGSERIAL PRIMARY KEY,
    entity_id    UUID NOT NULL,
    source_node  VARCHAR(64) NOT NULL,
    target_node  VARCHAR(64),            -- NULL = broadcast
    delta        JSONB NOT NULL,
    created_at   TIMESTAMPTZ DEFAULT now(),
    acked_at     TIMESTAMPTZ             -- NULL = pending
);

CREATE INDEX idx_delta_pending ON crdt_delta_log(target_node, created_at)
WHERE acked_at IS NULL;

-- Fetch pending deltas for a node
SELECT delta_id, entity_id, delta
FROM crdt_delta_log
WHERE (target_node = $1 OR target_node IS NULL)
  AND acked_at IS NULL
ORDER BY created_at
LIMIT 100;

-- Acknowledge receipt
UPDATE crdt_delta_log
SET acked_at = now()
WHERE delta_id = ANY($1);

-- Compact old deltas (after full sync)
DELETE FROM crdt_delta_log
WHERE acked_at < now() - interval '7 days';
```

---

## Implementation Examples

### G-Counter (Python)

```python
from dataclasses import dataclass, field
from typing import Dict

@dataclass
class GCounter:
    """Grow-only counter CRDT."""
    node_id: str
    counts: Dict[str, int] = field(default_factory=dict)

    def increment(self, amount: int = 1) -> Dict[str, int]:
        """Increment and return delta."""
        self.counts[self.node_id] = self.counts.get(self.node_id, 0) + amount
        return {self.node_id: amount}  # delta

    def value(self) -> int:
        return sum(self.counts.values())

    def merge(self, other: 'GCounter') -> None:
        """Merge remote state into local."""
        for node, count in other.counts.items():
            self.counts[node] = max(self.counts.get(node, 0), count)

    def merge_delta(self, delta: Dict[str, int]) -> None:
        """Merge delta (partial state)."""
        for node, count in delta.items():
            self.counts[node] = max(self.counts.get(node, 0), count)

    def state(self) -> Dict[str, int]:
        """Export full state for sync."""
        return dict(self.counts)

# Usage
counter_a = GCounter("node_a")
counter_b = GCounter("node_b")

delta = counter_a.increment(5)  # {node_a: 5}
counter_b.merge_delta(delta)    # Sync delta

counter_b.increment(3)
counter_a.merge(counter_b)      # Full sync

assert counter_a.value() == 8
assert counter_b.value() == 8
```

### LWW-Register (Python)

```python
from dataclasses import dataclass
from typing import Generic, TypeVar, Optional, Tuple

T = TypeVar('T')

@dataclass
class LWWRegister(Generic[T]):
    """Last-Writer-Wins Register with Lamport clock."""
    node_id: str
    value: Optional[T] = None
    clock: int = 0

    def set(self, value: T, clock: Optional[int] = None) -> Tuple[T, int, str]:
        """Set value and return delta (value, clock, node_id)."""
        self.clock = clock if clock is not None else self.clock + 1
        self.value = value
        return (self.value, self.clock, self.node_id)

    def get(self) -> Optional[T]:
        return self.value

    def merge(self, value: T, clock: int, node_id: str) -> bool:
        """Merge remote state. Returns True if local state changed."""
        if clock > self.clock or (clock == self.clock and node_id > self.node_id):
            self.value = value
            self.clock = clock
            return True
        return False

    def state(self) -> Tuple[Optional[T], int, str]:
        return (self.value, self.clock, self.node_id)

# Usage
reg_a = LWWRegister[str]("node_a")
reg_b = LWWRegister[str]("node_b")

delta = reg_a.set("hello")
reg_b.merge(*delta)

# Concurrent writes
reg_a.set("world", clock=5)
reg_b.set("earth", clock=5)

# After merge, "world" wins (node_a < node_b lexicographically? depends on comparison)
# If node_b > node_a: "earth" wins. Deterministic either way.
```

### OR-Set (Python)

```python
from dataclasses import dataclass, field
from typing import Dict, Set, Tuple, Generic, TypeVar

T = TypeVar('T')
Dot = Tuple[str, int]  # (node_id, clock)

@dataclass
class ORSet(Generic[T]):
    """Observed-Remove Set with dot-based tombstones."""
    node_id: str
    clock: int = 0
    entries: Dict[Dot, T] = field(default_factory=dict)
    context: Dict[str, int] = field(default_factory=dict)  # max clock per node

    def add(self, element: T) -> Dict:
        """Add element, return delta."""
        self.clock += 1
        dot = (self.node_id, self.clock)
        self.entries[dot] = element
        self.context[self.node_id] = self.clock
        return {"add": {dot: element}, "context": {self.node_id: self.clock}}

    def remove(self, element: T) -> Dict:
        """Remove element, return delta."""
        removed_dots = []
        for dot, e in list(self.entries.items()):
            if e == element:
                removed_dots.append(dot)
                del self.entries[dot]
        return {"remove": removed_dots}

    def lookup(self, element: T) -> bool:
        return element in self.entries.values()

    def elements(self) -> Set[T]:
        return set(self.entries.values())

    def merge(self, other: 'ORSet[T]') -> None:
        """Merge remote state."""
        # Keep entries not dominated by other's context
        for dot, element in other.entries.items():
            node, clock = dot
            if dot not in self.entries:
                # Only add if not already removed locally
                local_max = self.context.get(node, 0)
                if clock > local_max:
                    self.entries[dot] = element

        # Remove entries dominated by other's context
        for dot in list(self.entries.keys()):
            node, clock = dot
            if clock <= other.context.get(node, 0) and dot not in other.entries:
                del self.entries[dot]

        # Merge contexts
        for node, clock in other.context.items():
            self.context[node] = max(self.context.get(node, 0), clock)

# Usage
set_a = ORSet[str]("node_a")
set_b = ORSet[str]("node_b")

set_a.add("item1")
set_a.add("item2")
set_b.merge(set_a)

set_a.remove("item1")
set_b.add("item1")  # Concurrent add

set_a.merge(set_b)
set_b.merge(set_a)

# item1 survives (add-wins)
assert set_a.lookup("item1") == True
assert set_b.lookup("item1") == True
```

### Sync Service (Python)

```python
import asyncio
import json
from dataclasses import dataclass
from typing import Dict, Any, List
from aiohttp import ClientSession

@dataclass
class CRDTSync:
    """Delta-based CRDT synchronization service."""
    node_id: str
    peers: List[str]
    delta_buffer: Dict[str, List[Dict]] = field(default_factory=dict)
    vector_clock: Dict[str, int] = field(default_factory=dict)

    async def broadcast_delta(self, entity_id: str, delta: Dict[str, Any]):
        """Send delta to all peers asynchronously."""
        self.vector_clock[self.node_id] = self.vector_clock.get(self.node_id, 0) + 1

        message = {
            "entity_id": entity_id,
            "source": self.node_id,
            "clock": self.vector_clock.copy(),
            "delta": delta
        }

        async with ClientSession() as session:
            tasks = [
                self._send_delta(session, peer, message)
                for peer in self.peers
            ]
            await asyncio.gather(*tasks, return_exceptions=True)

    async def _send_delta(self, session: ClientSession, peer: str, message: Dict):
        """Send delta to single peer with retry."""
        for attempt in range(3):
            try:
                async with session.post(
                    f"{peer}/crdt/delta",
                    json=message,
                    timeout=5
                ) as resp:
                    if resp.status == 200:
                        return
            except Exception:
                await asyncio.sleep(0.1 * (2 ** attempt))

        # Buffer for later sync
        self.delta_buffer.setdefault(peer, []).append(message)

    async def receive_delta(self, message: Dict) -> Dict[str, Any]:
        """Process incoming delta, return ack."""
        entity_id = message["entity_id"]
        source = message["source"]
        remote_clock = message["clock"]
        delta = message["delta"]

        # Update vector clock
        for node, clock in remote_clock.items():
            self.vector_clock[node] = max(self.vector_clock.get(node, 0), clock)

        return {
            "status": "ok",
            "entity_id": entity_id,
            "clock": self.vector_clock.copy()
        }

    async def request_full_sync(self, peer: str, entity_id: str) -> Dict:
        """Request full state when delta sync fails."""
        async with ClientSession() as session:
            async with session.get(
                f"{peer}/crdt/state/{entity_id}"
            ) as resp:
                return await resp.json()
```

---

## Decision Trees

### 1. Choosing CRDT Type

```
What are you storing?
├── Single value (string, JSON, config)
│   ├── Conflicts should be auto-resolved?
│   │   └── YES → LWW-Register
│   └── Conflicts need manual resolution?
│       └── MV-Register
├── Numeric counter
│   ├── Increment only?
│   │   └── G-Counter
│   └── Increment and decrement?
│       └── PN-Counter
├── Collection of items
│   ├── Add only (grow-only)?
│   │   └── G-Set
│   ├── Add and remove, no re-add?
│   │   └── 2P-Set
│   └── Add, remove, and re-add?
│       └── OR-Set
├── Key-value map
│   └── OR-Map (OR-Set of (key, LWW-Register) pairs)
└── Ordered sequence (text)
    └── RGA or YATA (complex, use library)
```

### 2. State vs Delta Decision

```
Network bandwidth constrained?
├── NO → State-based (simpler)
└── YES
    ├── State size < 1KB?
    │   └── State-based (overhead not worth it)
    └── State size > 1KB?
        └── Delta-state
            ├── Can buffer deltas reliably?
            │   └── YES → Pure delta sync
            └── NO → Delta + periodic full sync fallback
```

### 3. Clock Type Selection

```
Multi-datacenter deployment?
├── NO → Lamport timestamp (simplest)
└── YES
    ├── Need human-readable timestamps for debugging?
    │   └── HLC (Hybrid Logical Clock)
    └── Pure causality tracking sufficient?
        └── Lamport timestamp

NTP synchronized within 10ms?
├── YES → Physical timestamp with node_id tie-breaker acceptable
└── NO → Must use logical clocks (Lamport or HLC)
```

### 4. Consistency Requirement Mapping

```
Consistency need → CRDT suitability
├── Linearizable (strong) → NOT suitable, use locks/consensus
├── Sequential consistency → NOT suitable
├── Causal consistency → CRDTs with vector clocks
├── Eventual consistency → All CRDTs
└── Strong eventual → CRDTs (guaranteed convergence)
```

---

## Performance Characteristics

### Space Complexity

| CRDT Type | State Size | Delta Size |
|-----------|------------|------------|
| G-Counter | O(nodes) | O(1) |
| PN-Counter | O(2 × nodes) | O(1) |
| LWW-Register | O(1) | O(1) |
| OR-Set | O(elements × adds) | O(1) per op |
| MV-Register | O(concurrent writes) | O(1) per write |

### Time Complexity

| Operation | G-Counter | LWW-Register | OR-Set |
|-----------|-----------|--------------|--------|
| Read | O(nodes) | O(1) | O(1) |
| Write | O(1) | O(1) | O(1) |
| Merge | O(nodes) | O(1) | O(entries) |

### Tombstone Management (OR-Set)

OR-Set tombstones can grow unbounded. Mitigation:

```python
def compact_tombstones(or_set: ORSet, retention_days: int = 7):
    """Remove old tombstones after all replicas synchronized."""
    cutoff = time.time() - (retention_days * 86400)

    # Only safe if all replicas have synced past this point
    for dot in list(or_set.context.keys()):
        if dot_timestamp(dot) < cutoff:
            # Verify all peers have this in their context
            if all_peers_synced(dot):
                del or_set.context[dot]
```

### Merge Performance Tips

```python
# BAD: Merge on every remote message
def handle_message(msg):
    crdt.merge(msg.state)  # O(n) per message

# GOOD: Batch merges
def handle_messages(messages):
    combined = reduce(merge_states, [m.state for m in messages])
    crdt.merge(combined)  # O(n) once

# BETTER: Delta batching with periodic full sync
def sync_loop():
    while True:
        deltas = collect_deltas(timeout=100ms)
        if deltas:
            merged_delta = reduce(merge_deltas, deltas)
            crdt.merge_delta(merged_delta)

        if time_since_full_sync > 1h:
            full_state = fetch_full_state(random_peer)
            crdt.merge(full_state)
```

---

## Production Checklist

### Before Deployment

- [ ] Confirm eventual consistency is acceptable for use case
- [ ] Define conflict resolution semantics (LWW, add-wins, etc.)
- [ ] Implement clock synchronization (Lamport minimum, HLC preferred)
- [ ] Set up delta sync with full-sync fallback
- [ ] Plan tombstone compaction strategy
- [ ] Define node_id assignment (UUID, hostname, etc.)

### Monitoring

```promql
# Sync lag between replicas
crdt_sync_lag_seconds{entity_type="counter"}

# Delta buffer size (indicates sync failures)
crdt_delta_buffer_size{node="node_a"}

# Merge rate
rate(crdt_merge_total[5m])

# Conflict rate (MV-Register)
rate(crdt_conflicts_total[5m])
```

### Alerting Rules

```yaml
groups:
  - name: crdt_alerts
    rules:
      - alert: CRDTSyncLag
        expr: crdt_sync_lag_seconds > 60
        for: 5m
        annotations:
          summary: "CRDT sync lag exceeds 60s"

      - alert: CRDTDeltaBufferHigh
        expr: crdt_delta_buffer_size > 1000
        for: 10m
        annotations:
          summary: "Delta buffer growing, sync may be failing"
```

---

## References

- [Delta-enabled CRDTs (Baquero)](https://github.com/CBaquero/delta-enabled-crdts) - Reference implementations
- [State-based CRDTs (Sypytkowski)](https://www.bartoszsypytkowski.com/the-state-of-a-state-based-crdts/) - Excellent tutorial series
- [A Comprehensive Study of CRDTs (Shapiro et al.)](https://hal.inria.fr/inria-00555588/document) - Academic foundation
- [CRDT Dictionary (Duncan)](https://www.iankduncan.com/engineering/2025-11-27-crdt-dictionary/) - Practical guide
- [Riak DT](https://github.com/basho/riak_dt) - Production-grade Erlang implementations
- [Automerge](https://automerge.org/) - JSON CRDT for collaborative editing
- [Yjs](https://yjs.dev/) - High-performance CRDT for text collaboration

---

## Glossary

| Term | Definition |
|------|------------|
| **CRDT** | Conflict-free Replicated Data Type - data structures that merge without coordination |
| **CvRDT** | Convergent (state-based) CRDT - syncs by sending full state |
| **CmRDT** | Commutative (operation-based) CRDT - syncs by sending operations |
| **δ-CRDT** | Delta-state CRDT - syncs by sending state deltas |
| **Dot** | Unique identifier for an operation: (node_id, logical_clock) |
| **Join semi-lattice** | Mathematical structure where merge is commutative, associative, idempotent |
| **Tombstone** | Marker for deleted elements, required for distributed deletion |
| **Vector clock** | Logical clock tracking causality across multiple nodes |
| **Lamport timestamp** | Simple logical clock: max(local, received) + 1 |
| **HLC** | Hybrid Logical Clock - Lamport clock bounded by physical time |
| **Add-wins** | Conflict resolution where concurrent add beats remove |
| **LWW** | Last-Writer-Wins - conflict resolution by timestamp |

---

**Version:** 1.0
**Last Updated:** 2026-02-04
