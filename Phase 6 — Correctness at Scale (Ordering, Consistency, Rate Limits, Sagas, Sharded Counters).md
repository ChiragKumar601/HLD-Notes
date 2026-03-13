
## Outcome of Phase 6

You should be able to:
- reason about **CAP/consistency** with real design choices
- design **unique IDs** + ordering guarantees
- implement-safe patterns for **duplicates/retries**
- add **rate limiting** correctly (placement + algorithm)
- design **multi-step workflows** using sagas/outbox
- solve **hot key** counters with sharding



# 6.1 CAP + Consistency Models (practical, interview-ready)

## 6.1.1 Intuition

In distributed systems, the network can partition. When it does, you choose between:
- **Consistency (C):** every read sees latest write
- **Availability (A):** every request gets a non-error response
- **Partition tolerance (P):** system continues despite network split (must-have). In a distributed system, you must assume **network partitions** can happen (temporary split, packet loss, one AZ/region can’t talk to another). So **P (partition tolerance) is mandatory**.

So the real decision is: **under partition, pick CP or AP

```
Partition happens:
Node1  X  Node2

CP: reject/timeout some requests to keep correctness
AP: accept requests, reconcile later (eventual)
```

### One-liner
- **Lose C:** responses may be **stale/temporarily inconsistent**.
- **Lose A:** some requests **error/timeout** (service becomes partially unavailable) to stay correct.

## 6.1.2 Concepts

- **Strong/linearizable Consistency**: reads reflect latest committed write.
	- If a write succeeds, any later read will see it.
	- Feels like “one single up-to-date copy”.
	Use when correctness is critical:
	- payments, ledger, inventory finalization, auth/security settings.

- **Eventual Consistency**: System may return stale data temporarily, but replicas will converge over time. Great for scale and availability. Use when slight staleness is fine:
	- feeds, like counts, view counters, analytics.

- **Causal Consistency**: - Preserves “happened-before” ordering. If you saw A, you should not later see B that depends on A missing.
	Useful in:
	- chat/comments (“I replied to your message” shouldn’t appear before your message).

**When to choose what**
- Money/ledger: CP-ish behavior for critical paths
- Feeds/likes: AP-ish, eventual is fine
- User profile: often eventual ok; strong for security settings/password changes

## 6.1.3 Architecture implication

**If you want strong consistency**
You typically need:
- **leader-based writes** or **quorum writes/reads**
- and under partition you may:
    - reject writes in the “minority” side
    - return errors/timeouts rather than risk wrong state

So: more correctness, potentially less availability under partition.

**If you allow eventual consistency**
You need:
- conflict handling (e.g., last-write-wins, merge rules, CRDTs)
- reconciliation processes (read repair, background sync)
- idempotency/dedupe (because retries happen)

So: more availability, but you must handle “two sides wrote different things”.

- **Strong consistency:** “Write only via leader/quorum; minority rejects → correct but may be unavailable during partitions.”
- **Eventual consistency:** “Accept writes everywhere; resolve conflicts later → always responsive but can be temporarily inconsistent.”

## 6.1.4 Practical example
“Like count” can be stale; “payment status” cannot.

## 6.1.5 Real-world use cases
- geo-replicated databases, multi-region active-active systems

## 6.1.6 Interview questions

**Q. For chat messages, what consistency do you need and why?**
For chat, I want causal / per-conversation ordered consistency so ‘happened-before’ is preserved (message before reply/edit), while allowing eventual consistency across unrelated conversations.
 
**Q. Under partition, would you rather drop writes or serve stale reads?**
For **CP-critical paths** (payments/auth/inventory), I’d **reject writes/requests** when I can’t reach leader/quorum (503/timeout) rather than risk inconsistency; for **AP paths** (feeds/likes), I’d keep serving from replicas/cache even if stale and reconcile later.


# 6.2 Sequencer / Unique ID Generator (ordering + uniqueness)

## 6.2.1 Intuition

In real systems, IDs often act like:
- **a timestamp** (“when did it happen?”)
- **an ordering key** (“which came first?”)
- **a routing/sharding hint** (“which shard should store this?”)
- **a correlation handle** (“trace this event across services”)

So the ID strategy affects:
- DB index performance
- pagination
- ordering guarantees
- scalability

## 6.2.2 Concepts (must-know)

Common ID strategies:

- **Auto-increment (single DB)**:  Example: `order_id BIGSERIAL`
	✅ Pros
	- simple
	- ordered
	- great DB index locality
	
	❌ Cons
	- hard to generate globally across many shards/regions
	- can become bottleneck if you need one place to assign IDs
	- sequential IDs can leak business volume (predictable)
	
	**Best for:** single-region single-primary DB, smaller systems.

- **UUIDv4**: Example: `550e8400-e29b-41d4-a716-446655440000`
 	✅ Pros
	- globally unique without coordination
	- great for distributed generation
	
	❌ Cons
	- not ordered → DB index gets fragmented (random inserts across B-tree)
	- worse for “latest N” ordering unless you store separate `created_at`
	- bigger storage than bigint

	**Best for:** distributed systems where uniqueness matters more than ordering

- **Snowflake-style**: This is a common “best of both worlds” approach.
	- timestamp gives **rough ordering**
	- workerId makes it unique across nodes
	- sequence handles multiple IDs within the same millisecond
	✅ Pros
	- scalable (no central DB needed per ID)
	- ordered-ish (good index locality vs UUIDv4)
	- supports “sort by id” often
	❌ Cons
	- requires workerId assignment (coordination)
	- needs careful handling of **clock drift/backwards time**
	- ordering is “time-based”, not perfect across nodes if clocks drift
	**Best for:** high-scale systems (orders, events) where you want sortable IDs.

- **DB-based allocator**: Instead of generating every ID centrally, you allocate **ranges**.
	Example:
	- service asks DB: “give me next block of 10,000 IDs”
	- DB returns [1,000,001 … 1,010,000]
	- service uses them locally
	✅ Pros
	- ordered IDs
	- reduced contention vs per-ID allocation
	- simple correctness model
	❌ Cons
	- still some coordination (block allocation)
	- gaps can happen if service crashes with unused IDs

	**Best for:** when you want integer IDs and can tolerate gaps.
### Snowflake-ish layout (example)
```
[ timestamp ][ machineId ][ sequence ]
```

## 6.2.3 Architecture

```
Services -> ID Service (or embedded generator)
             |
           (lease workerIds, track time drift)
```

Key concerns:
- clock drift/backwards time
- workerId assignment (coordination)
- monotonicity per shard vs globally

- **ID service:** a central service generates IDs for everyone—simpler for clients, but adds an extra dependency/bottleneck.
- **Embedded generator:** each instance generates IDs locally—no network hop, but you must safely coordinate worker IDs and time.

## 6.2.4 Practical example

Chat messages: per-conversation ordering key = (conversationId, messageSeq)

## 6.2.5 Real-world use cases
- order IDs, message IDs, event IDs

## 6.2.6 Interview questions

**Q. What happens if clock goes backward in Snowflake?**
It can generate **non-monotonic or duplicate IDs** (timestamp part decreases), which can break ordering; generators typically **wait until time catches up** or use a **logical sequence** to stay monotonic.

**Q. Do you need total global ordering or per-key ordering?**
Most systems only need **per-key ordering** (e.g., per `conversationId`/`orderId`) to scale; total global ordering is expensive and usually unnecessary except for strict ledgers/audit logs.


# 6.3 Rate Limiter (protect systems, prevent abuse)

## 6.3.1 Intuition

Rate limiting is **system self-defense**:
- stops abuse
- prevents overload
- enforces fairness (per-user/per-tenant)

## 6.3.2 Concepts (must-know)

Placement:
- **Edge/API Gateway:** enforce global per-client/tenant/IP limits early to stop abuse before it hits services.
- **Service-level:** protect specific services/dependencies by limiting expensive internal endpoints close to the code.
- **Per-resource:** apply fine-grained limits on hot entities (e.g., per userId/orderId) when needed, at higher state/complexity cost.

Algorithms:
- **Token bucket** (best default: allows bursts):
	Think of a bucket that holds **tokens**.
	**Setup**
	- Bucket capacity = **5 tokens**
	- Refill rate = **0.5 token/sec** (so 5 tokens per 10 seconds)
	**Rule:** each request costs **1 token**.
	- If token available → allow and subtract 1
	- If no tokens → reject (or wait)
	
	What it “feels” like
	If you were idle, the bucket fills to 5, so you can **burst 5 requests instantly**.
	Example:
	- At t=0, you have 5 tokens → you send 5 requests immediately ✅ (tokens go to 0)
	- Now refill is 0.5/sec → after 2 seconds you have 1 token → you can send 1 more ✅
	**Key idea:** Token bucket enforces the **average rate**, but allows **bursts** up to bucket size.

- **Leaky bucket** (smooth output, no bursts):
	Leaky bucket is about making traffic leave at a **steady rate**.
	**Setup**
	- Leak rate = **0.5 req/sec** (same “5 per 10s” rate)
	- There’s a small queue/bucket capacity (say 5) to hold pending requests
	**Rule:**
	- requests arrive and go into a bucket/queue
	- they “leak out” to downstream at constant rate
	- if bucket is full → reject new requests (or drop)
	 
	What it “feels” like
	If 5 requests come at once:
	- they get queued
	- downstream sees them processed smoothly over ~10 seconds (0.5/sec)
	**Key idea:** Leaky bucket smooths load; it doesn’t allow bursty downstream behavior.


- **Fixed window** (simple, boundary spikes):
	Rule: “You can buy **60 tickets per hour**.”
	- Counter resets at the top of the hour.
	- Problem: you can buy 60 tickets at **12:59** and another 60 at **1:00** → 120 tickets in 1 minute.
	✅ Very simple  
	❌ Boundary spike problem

- **Sliding window**:
	 Rule: “At any moment, you can buy at most **60 tickets in the last 60 minutes**.”
	- It doesn’t reset at a boundary.
	- If you already bought 60 recently, you must wait until some of those purchases fall out of the last-60-min window.
	✅ Most accurate fairness  
	❌ More state/compute

## 6.3.3 Architecture

```
Client -> Gateway -> RateLimiter (Redis/KV) -> Service
```

- Gateway checks limiter before forwarding.
- Limiter state is stored in **Redis/KV** so all gateway instances share the same counters/tokens.
- Often you add **local caching** to reduce Redis calls.

### Key design choices
- **Key selection:** what you limit by
    - `tenantId`, `userId`, `apiKey`, `IP`
- **Distributed store:** Redis is common (fast atomic ops)
- **Fail-open vs fail-closed:**
    - fail-open: allow traffic if limiter is down (better availability, risk overload)
    - fail-closed: block traffic if limiter is down (protect system, may hurt users)

## 6.3.4 Practical example

Per-tenant API limit: 100 rps burst 200.
- token bucket keyed by tenantId

## 6.3.5 Real-world use cases
- **Login/OTP**: prevent brute-force attacks
- **Public APIs**: fairness for customers
- **Webhooks**: protect you from misbehaving partners that retry too aggressively


## 6.3.6 Interview questions

**Q. Fail-open vs fail-closed: which for login? which for payments?**
- **Login:** usually **fail-closed** for the _rate limiter_ to prevent brute-force/OTP abuse; allow a safe fallback only if you have other protections (WAF/IP reputation).
- **Payments:** **fail-closed** (protect correctness/cost and prevent abuse).

**Q. How do you avoid the rate limiter becoming a bottleneck?**


# 6.4 Exactly-once effects: Idempotency + Dedup + Outbox

## 6.4.1 Intuition

You can’t prevent duplicates; you design so duplicates do nothing harmful.
Your design goal becomes: **duplicate processing should not cause double charge / double email / double inventory deduction.**

## 6.4.2 Concepts

- **Idempotency keys** on write APIs
- **Dedupe table** on consumers (`event_id` seen)
- **Transactional outbox** to safely publish events after DB commit

## 6.4.3 Architecture (outbox pattern)

```
Service writes:  
  DB transaction:  
    - business row update  
    - outbox row insert  
  
Outbox relay:  
  reads outbox -> publishes -> marks sent
```

Write the business change and an outbox event row in the same DB transaction, then a background relay publishes outbox events to the broker and marks them sent—so there’s no ‘DB committed but event lost’ half state.
## 6.4.4 Practical example
Order created + OrderPlaced event published reliably.

## 6.4.5 Real-world use cases
- payment processing, notifications, inventory workflows


## 6.4.6 Interview questions

**Q. What bug happens if you publish event before DB commit?**
If you publish before the DB commit, consumers can act on an event for data that later **rolls back or isn’t visible yet**, creating **phantom side effects** (like emails/charges for an order that doesn’t exist)

**Q. How do you dedupe if your consumer is stateless?**
Use an external **durable dedupe store** keyed by `eventId/idempotencyKey`  (DB table with a unique constraint; optionally Redis as a fast TTL cache); on consume, **insert-if-not-exists** then apply side effect and ack.


# 6.5 Distributed Transactions + Sagas (multi-step workflows)

## 6.5.1 Intuition

In monolith + one DB, you can do one ACID transaction:
- reserve inventory
- create order
- charge payment      
…all in one commit/rollback.

In microservices, inventory, payments, shipping often have **different databases**. So you can’t easily do one atomic transaction across them.

**Saga** is the common solution:
- each service does a **local transaction**
- system reaches **eventual consistency**
- if something fails, you run **compensations** to undo earlier steps.

## 6.5.2 Concepts

* ***2PC (Two-Phase Commit):** A coordinator first asks all participants to **prepare/lock** resources, then issues a single **commit/abort** so it’s all-or-nothing—strong consistency but **blocking and fragile** under failures.
* ***Saga:** A distributed workflow of **local transactions**; if a later step fails, you run **compensating actions** to undo earlier steps—**eventual consistency** without global locking.
	- **Orchestrated Saga:** A central **orchestrator** drives the steps and triggers compensations—easier to reason about, but adds a coordinator component.
	- **Choreography Saga:** No central brain; services **publish/consume events** and react to each other—more decoupled, but harder to trace and can become event-spaghetti.

**Compensation**: Compensation is an **undo-like action**.
Important: compensation is not always a perfect undo (real world), but it restores business correctness.

## 6.5.3 Architecture (orchestrated saga)

```
Order Service -> Saga Orchestrator  
   1) reserve inventory  
   2) authorize payment  
   3) confirm order  
fail -> compensate:  
   - release inventory  
   - void payment auth
```

## 6.5.4 Practical example

Checkout flow with inventory + payment + shipping.

## 6.5.5 Real-world use cases
- ecommerce checkout, travel booking, fintech workflows

## 6.5.6 Interview questions

**Q. What makes a good compensation action?**
“A good compensation isn’t a perfect undo; it restores a valid business state and is **idempotent/retry-safe** (can be run multiple times without harm).

**Q. How do you handle a “stuck saga” (service down mid-flow)?**
Handle a stuck saga by persisting saga state and using **timeouts + retries**; if it still can’t progress by a deadline, **trigger compensations** for completed steps and push it to **DLQ/alerting** for manual recovery—keeping all actions idempotent.


# 6.6 Sharded Counters (hot key scaling)

## 6.6.1 Intuition

A single counter key becomes a hotspot at high QPS.  
Sharded counters spread load across multiple keys, then aggregate.

If everyone likes the same post, and you store count in one key:
- `likes:post123`

Then every like is an `INCR` on the **same key** → one Redis shard/CPU becomes the bottleneck (**hot key hotspot**).

So you **split** the counter into many keys (shards) so writes spread out.

## 6.6.2 Concepts

**Write path:**
- increment one shard: `counter:post123:shard7`  
 Instead of incrementing one key, you increment **one of many** keys:
- `counter:post123:shard0`
- `counter:post123:shard1`
- …
- `counter:post123:shard7`

To choose shard:
- random shard
- or `hash(userId) % N` (better distribution per user)

Each write hits a different shard key → load spreads.


**Read path:**
To show total likes/views:
- **sum all shards** on read  
    (read `N` keys and add)  
    OR
- maintain a **cached aggregate** value updated periodically (faster reads)

Trade-offs:
- reads become more expensive or slightly stale
- write becomes scalable
- it’s a write-scalability vs read-cost/freshness trade-off.

## 6.6.3 Architecture

```
Writers -> choose shard -> KV/Redis increments  
Readers -> sum shards OR read cached aggregate  
Background job -> periodically compacts/aggregates
```

## 6.6.4 Practical example
A post goes viral: 50k likes/sec.  
Single key approach breaks. Sharded counters:
- 32 shards
- each shard gets ~1/32 of traffic
- aggregator updates total every 1–5 seconds

User sees “1,234,567 likes” maybe a second behind—acceptable for likes/views.

## 6.6.5 Real-world use cases
- likes/views/download counts
- analytics counters
- internal rate limiter counters at very high QPS

## 6.6.6 Interview questions

**Q. How many shards would you choose and why?**
Pick **N shards** so **per-shard QPS** stays below what one Redis key/partition can handle with headroom: `N ≈ peakQPS / targetQPSperShard` (then round up, e.g., 16/32/64). Start modest and increase if a shard becomes hot.
 
**Q. How do you make reads fast without summing 100 shards each time?**
Maintain a **cached total** updated by a periodic/background aggregator (or on-write batching), so reads hit `counter:post123:total` and accept slight staleness.
 


# Phase 6 Drill (25 minutes)

**Prompt:** “Design a payment checkout system.”  
Must include:
- consistency requirements (what must be CP-like)
- idempotent APIs + dedupe
- saga for multi-step flow
- rate limiting for abuse
- unique IDs for orders/payments
- failure handling + reconciliation jobs