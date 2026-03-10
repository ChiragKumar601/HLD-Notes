
### Outcome of Phase 3

You should be able to:
- pick the right database type for a workload
- design a schema from access patterns (with indexes)
- explain replication vs partitioning trade-offs
- sketch a scalable KV store (a classic HLD building block)

# 3.1 Databases: how to choose (interview decision tree)

## Intuition

A database is a compromise between **correctness, latency, throughput, cost, and complexity**.  
In interviews, you must show you can choose based on **access patterns + constraints**, not hype.

## Concepts (must-know)

### A) Types of databases (when to use which)

**Relational (OLTP)**
Best for **core truth** and correctness:
- ACID transactions, joins, constraints
- Examples: orders, payments, inventory

**Document DB**
Best for **flexible/nested** data:
- JSON-like documents, schema flexibility
- Examples: user profiles, content metadata

**Key-Value**
Best for **ultra-fast “get by key”**:
- Examples: sessions, tokens, rate limit counters, config
- Redis/Dynamo KV patterns

 **Wide-column (Cassandra-like)**
 Best for **huge scale, write-heavy, time-series-like** workloads:
- Examples: logs/events at massive scale, IoT data  

 **Search index (Elastic/OpenSearch)**
 Best for **text search + ranking + filters**:
- product search, autocomplete

**OLAP / Data warehouse (BigQuery/Snowflake/Redshift)**
Best for **analytics**:
- large scans, aggregates, reporting dashboards

**Time-series DB (Prometheus/Influx)**
Best for **metrics over time**:
- CPU, latency, QPS tracking

Graph DB
Best for **relationship traversals**:
- “friends of friends”, recommendations, fraud networks

**Interview rule:** Keep one DB (usually OLTP relational) as **source of truth**, and build others (search/cache/warehouse) as **derived** from it.


### B) Replication (scale reads + availability)
Replication means keeping **multiple copies of the same data** on different machines so you get **high availability** and **more read capacity**.

- **Leader–follower (primary–replica)**
    - writes go to leader, reads can go to replicas
    - trade-off: **replica lag** (eventual consistency on replicas)
- **Multi-leader**
    - writes in multiple regions → conflict resolution needed
- **Quorum replication (N replicas, R/W quorums)**
    - tune consistency vs latency with (R, W)
**Practical interview phrasing**
- “Strong reads” → read from leader (or quorum)
- “Cheap reads” → read from replicas (accept staleness)


### C) Partitioning / Sharding (scale writes + storage)

Partitioning distributes data across nodes.

**Strategies**
- **Hash by key** (uniform distribution)  
    Good: KV store, sessions  
    Risk: range queries become hard

- **Range partition** (by time/id range)  
    Good: time-series, logs  
    Risk: hot partitions on “latest range”    

- **Directory / consistent hashing ring**  
    Used in KV systems

**Hot key problem**
- One partition gets 90% traffic (celebrity user, viral URL)  
    Mitigations: caching, sharding the hot key, request coalescing.


### D) Common trade-offs you must say out loud

- Strong consistency vs latency/availability under partition (CAP reality)
- Normalization vs denormalization
- Secondary indexes: faster reads, slower writes
- Joins vs precomputed views
- Multi-region writes: availability vs conflict complexity


## Architecture (DB in a typical service)

```
          +------------------+
Client -> | API / Service    |
          +------------------+
                 |
                 | writes
                 v
          +------------------+
          | DB Leader        |
          +------------------+
             |          |
        repl |          | repl
             v          v
        +---------+  +---------+
        | Replica |  | Replica |
        +---------+  +---------+
```

## Practical example

E-commerce:
- Orders + payments in **relational**
- Product search in **search index**
- Sessions in **KV / Redis**
- Analytics in **warehouse**

## Interview questions

**Q. When do you read from replicas vs leader?**
Read from the **leader** when you need **fresh/strong read-after-write correctness** (e.g., payments), and from **replicas** for **read-heavy, latency/cost-optimized** paths that can tolerate **replica lag**.
 
**Q. Hash vs range partition—what breaks in each?**
In case of Range partition, usually most traffic is on latest data- so the newest shard might get overloaded. In case of has partition range queries can become hard and might require scanning many shards.

**Q. How do you handle a hot shard?**
- **Caching:** serve hot reads from Redis/CDN
- **Sharding the hot key:** split one key into sub-keys (e.g.,`userId#1..#N`)
- **Request coalescing:** if 100 requests ask same thing, combine them so only 1 hits DB (avoid stampede)

# 3.2 Data modeling: start from access patterns (not ER diagrams)

## Intuition
Data modeling is reverse engineering:  
**Queries → schema → indexes → constraints → migrations**

## Concepts (must-know)

### A) The 6 questions that drive schema
1. What are top read queries? (by key? by user? by time?)
2. What are top write paths? (single row vs multi-row transaction?)
3. What needs strict correctness? (money, inventory)
4. What can be eventual? (counters, analytics)
5. What’s cardinality? (many-to-many?)
6. What’s growth/retention? (TTL, archiving)

### B) Indexing rules (interview-ready)

- Index for **WHERE + ORDER BY**
- Composite index order matters: `(user_id, created_at)` supports:
    - `WHERE user_id=? ORDER BY created_at`
- Avoid indexing low-cardinality fields (like boolean) unless combined
- Every index slows writes—be selective

### C) Versioning / optimistic locking (for concurrency)

Add:
- `version` int (or `updated_at`) and update with condition  
    Used for preventing lost updates.

### D) Denormalization (when acceptable)

Denormalize for:
- heavy read endpoints where joins hurt latency  
    But you pay:
- duplication + write complexity + backfills


## Practical example: “Orders for a user”

**Access patterns**
- Create order
- List user orders sorted by time
- Get order by id
- Update order status (monotonic)

**Schema sketch**

- `orders(order_id PK, user_id, status, total, created_at, version)`
- Index: `(user_id, created_at desc)`
- Index: `(status, created_at)` only if needed for ops dashboards/jobs

## Real-world use cases
- feeds, chat messages, ecommerce order history, audit logs.

## Interview questions

**Q. Design schema for “conversation messages” for fast recent reads.**

**Q. What composite index would you add for “last 20 orders by user”?**
a composite index on **`(user_id, created_at DESC)`**

**Q. When do you introduce a separate read model (CQRS-ish)?**
When we want read and right to be separated from each other


# 3.3 Key-Value Store (KV Store) — classic system design building block

### Intuition
KV stores are the simplest distributed datastore:
- put/get by key
- scale horizontally with sharding  
    They teach the fundamentals: partitioning, replication, versioning, failure detection.
    Examples: sessions, tokens, feature flags, rate limit counters.


## 3.3.1 Requirements (clarify first)

Functional:
- `PUT(key, value)`
- `GET(key)`
- optional: `DELETE`, `LIST(prefix)` (prefix makes it harder)

Non-functional:
These drive architecture:
- **Latency**: single-digit ms ⇒ data in memory, avoid cross-region calls.
- **Availability**: still works if some nodes are down ⇒ replication.
- **Scalability**: add nodes as data/QPS grows ⇒ sharding.
- **Consistency**: strong vs eventual ⇒ quorum + conflict handling.
- **Durability**: in-memory only vs persisted ⇒ WAL/snapshots.


## 3.3.2 High-level design

```
Clients
  |
  v
+-------------------+
| Router / Coordinator|
+-------------------+
   |   |   |
   v   v   v
+-----+  +-----+  +-----+
|Node |  |Node |  |Node |
+-----+  +-----+  +-----+
   |        |        |
  (replication between nodes)
```

But routers can be a bottleneck; common approach:
- clients use a **consistent hashing ring** to find the right nodes.


## 3.3.3 Partitioning (consistent hashing ring)

### Concept

Hash(key) → position on ring → choose next node(s).

```
  [N1]------[N2]
   |          |
   |          |
  [N4]------[N3]

key K hashes near N2 -> primary = N2, replicas = N3, N4
```


Benefits:
- Adding/removing nodes moves only a fraction of keys.

Mitigation: **virtual nodes** to balance load.


## 3.3.4 Replication (N replicas)

For each key:
- store on **N** nodes: primary + replicas

Choices:
- **Async replication**: fast writes, eventual consistency
- **Sync/quorum replication**: stronger consistency, higher latency

Quorum idea:
- N = 3 replicas
- Write requires W=2 acks
- Read requires R=2 to overlap → strong-ish reads

Consistent hashing maps **nodes and keys onto a hash ring** so each key is stored on the **next clockwise node** (and the next **N−1** nodes for replicas); when nodes are added/removed, only the affected ring segment’s keys move. Consistency is tuned with quorums: **W** replica acks for writes and **R** replicas read, where **R+W>N** gives strong-ish reads.

## 3.3.5 Versioning + conflicts

Conflicts happen when:
- replication is async
- multiple writers update the same key
- partitions cause different replicas to accept writes

You need a way to decide which value is “the latest”.
If eventual or multi-writer, you need conflict handling.

Options:
- **Last-write-wins** (timestamp) — simple, can lose updates
- **Vector clocks** — detects concurrent updates (complex)
- **App-level merge** — best when domain supports it

Interview-friendly: start with LWW; mention vector clocks for advanced.


## 3.3.6 Fault tolerance + failure detection

Problems:
- node down
- network partitions
- slow node (tail latency)

Mechanisms:
- Gossip/heartbeats :Nodes exchange membership info (“who is alive”).
- Reroute on ring :If a node is down, clients/routers pick next healthy replica.
- Hinted handoff: If a replica is down, temporarily store its write on another node (“hint”) and deliver later when it recovers.
- Read repair: On read, if replicas disagree, return the newest and also update stale replicas in background.
- Anti-entropy: Periodic background sync (like Merkle trees) to ensure replicas converge over time.



## 3.3.7 A concrete read/write flow (what to narrate)

### PUT(key, value)
1. compute nodes via hash ring
2. send to primary + replicas
3. wait for W acks
4. return success

### GET(key)
1. compute nodes
2. read from R replicas (or nearest replica)
3. reconcile version, optionally repair



### Practical example (where KV store is used)

- sessions, feature flags, caching layer (Redis-like), rate limiter counters, user settings.

### Real-world use cases
- high QPS, low latency lookups with simple access patterns.

### Interview questions

**Q. How do you prevent hotspots if one key is extremely popular?**
- We can shard that one hot key.
- We can store that key in cache memory(Redis).
- We can also do request coalescing.
- Add rate-limiting/ dedicated hot-shard/ replicate-read
 
**Q. How do you support `LIST(prefix)` efficiently?**
Maybe we can use a data structure like trie to store keys in the lexographical order, because we 'll need that for ordered index or maybe a secondary map for (prefix, key) service.
 
**Q. What changes when you need strong consistency?**
Strong consistency means you can’t rely on async replication/LWW; you need **leader or quorum writes/reads** (e.g., N replicas with **R+W>N**, or a single leader with sync replication) and accept higher latency and reduced availability during partitions.
 
## Phase 3 Drill (20 minutes)

**Prompt:** “Design a pastebin-like text storage service.”  
Must include:
- DB choice (truth store)
- schema + indexes from access patterns
- replication vs partition plan
- mention cache/CDN if reads dominate
- deep dive: hot partitions and retention/TTL