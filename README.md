# HLD-Notes

# Modern System Design (HLD) Course — SDE Interview Prep

A practical, implementation-friendly High Level Design (HLD) course structured around real FAANG-style System Design interview patterns.  
This repo is built to help you **lead** system design interviews like a strong senior level SDE: clarify requirements, estimate scale, choose the right building blocks, justify trade-offs, and design for reliability and operations.


## What you’ll learn
By the end of this course, you will be able to:
- Drive an interview using a consistent workflow (**requirements → numbers → APIs/data → HLD → deep dives → failures/ops → trade-offs**)
- Make correct choices around **consistency, availability, partition tolerance (CAP)** based on product needs
- Design scalable systems using core building blocks: **LBs, caches, queues, pub-sub, blob storage, search, KV stores**
- Build systems that survive production realities: **timeouts, retries, circuit breakers, backpressure, rollouts, DR**
- Communicate like an SDE-3: crisp assumptions, clear diagrams, explicit trade-offs

---

## Course Structure (7 Phases + Prerequisites)

### P0) Prerequisites (Foundations)
Covers the minimum foundation required to understand and apply HLD:
- Distributed system mental model (request lifecycle)
- HTTP/API essentials (status codes, idempotency, pagination, versioning)
- Networking basics (DNS, TCP/TLS, keep-alive, timeouts/retries)
- Data fundamentals (keys, indexes, transactions, isolation intuition)
- Concurrency basics (duplicates, idempotency, exactly-once effects)
- Linux/Debugging essentials (health checks, logs/metrics/traces)
- Cloud primitives (regions/AZs, managed building blocks)
- Interview-driving workflow (RESHADED + 7-step loop)

---

### Phase 1) Interview Core + Thinking Framework
Learn how to *lead* system design interviews:
- Requirements: functional + non-functional (SLO-ish thinking)
- Back-of-the-envelope calculations (QPS, storage, bandwidth, latency budgets)
- Decision-making and scope control
- Deep-dive selection strategy
- Interview deliverable template

---

### Phase 2) Network & Traffic Front Door
Design the entry path for traffic:
- DNS (TTL, routing, failover, geo/latency routing)
- Load balancers (L4/L7, algorithms, health checks, draining)
- API Gateway responsibilities (auth, rate limits, routing, observability)
- API contracts: errors, pagination, idempotency, versioning

---

### Phase 3) Databases + Data Modeling + KV Store
Design the truth layer and model data from access patterns:
- Database types and selection (OLTP, NoSQL, search, OLAP)
- Replication vs partitioning and trade-offs
- Indexing rules and schema decisions
- Designing a scalable Key-Value store:
  - consistent hashing, replication, quorum reads/writes
  - versioning/conflict resolution
  - failure detection, read repair, anti-entropy

---

### Phase 4) Performance Layer: Cache + CDN + Blob Store
Make systems fast and cost-effective:
- Distributed cache (cache-aside/read-through/write-through)
- TTLs, invalidation strategies, stampede protection
- CDN design (cache keys, purges, versioned assets, origin shielding)
- Blob store (metadata vs bytes, pre-signed uploads, multipart, lifecycle)

---

### Phase 5) Asynchrony: Queue + Pub/Sub + Streams
Build event-driven systems that scale:
- Distributed messaging queue (acks, retries, DLQ, visibility timeout)
- Pub-sub fanout patterns and subscriber isolation
- Streams vs queues (retention, replay, offsets, consumer groups)
- Exactly-once effects via idempotency + dedupe + transactional outbox

---

### Phase 6) Correctness at Scale
Handle the hard distributed systems problems:
- CAP and consistency models (strong/eventual/causal)
- Sequencers and ID generation (Snowflake-style, ordering needs)
- Rate limiting (token bucket, placement, fail-open vs fail-closed)
- Distributed transactions: sagas + compensations
- Sharded counters (hot key mitigation, aggregation strategies)

---

### Phase 7) Operability & SDE-3 Differentiators
Design systems that survive real production:
- Observability: logs/metrics/traces + correlation
- SLO/SLI/SLA and error budgets
- Resilience: timeouts, retries, backoff+jitter, circuit breakers, bulkheads
- Overload control: backpressure, shedding, bounded concurrency
- Security & privacy basics (AuthN/AuthZ, secrets, encryption, auditability)
- Safe rollouts and migrations (canary, blue/green, feature flags, expand/contract)
- DR + geo design (RPO/RTO, multi-AZ, active-passive vs active-active)


Recommendation - This course is designed to be read inside **Obsidian**. Clone the repo and open it as an **Obsidian Vault**. Each phase/module is a folder; each topic is a standalone note.
