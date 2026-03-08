
### Outcome of Phase 7
You should be able to:
- define **SLOs** and design around **error budgets**
- build systems that degrade gracefully under overload
- design **observability** (logs/metrics/traces) that actually helps on-call
- apply **timeouts/retries/circuit breakers** correctly
- cover **security + privacy** essentials in HLD
- explain safe **rollouts + migrations**
- design **multi-AZ / multi-region** with clear RPO/RTO trade-offs


# 7.1 Observability: logs, metrics, traces (and how they connect)

## 7.1.1 Intuition

If you can’t measure and debug your system quickly:
- you won’t know it’s failing,
- you won’t know why,
- and you won’t be able to fix it during an incident.

So in HLD, you always design observability alongside the architecture.

## 7.1.2 Concepts (must-know)

### A) Metrics (numbers over time)

Metrics answer: **“Is the system healthy?”**
Key metrics (golden signals):
- **Traffic (QPS):** how much load
- **Latency:** p50/p95/p99 response time
- **Errors:** 4xx vs 5xx rate
- **Saturation:** CPU, memory, thread pools, DB connections, queue backlog

Also track dependencies:
- DB latency + connection pool usage
- cache hit rate + cache latency
- queue lag/backlog + DLQ rate

### B) Logs (events with context)
- structured logs (JSON) so you can search/aggregate
- must include: `traceId/requestId`, service name, route, status, latency, key identifiers (orderId/tenantId)
- avoid logging secrets/**PII** (Personally Identifiable Information)(passwords, tokens, phone numbers, raw user data)

### C) Traces (one request across services)
Traces answer: **“Where did the time go?”**

A trace is a request’s journey:
- gateway → service A → service B → DB/cache  
    Each step is a **span** with timing.

Best use:
- diagnosing **p95/p99 tail latency**
- finding which dependency caused slowness

### D) Correlation (the key)
- traceId propagated through headers (`traceparent` / `x-request-id`)
- logs reference traceId → jump from metrics spike → trace → exact log line

## 7.1.3 Architecture

```
Service  
  |-- metrics --> TSDB --> dashboards/alerts  
  |-- logs -----> log store --> search/RCA  
  |-- traces ---> trace backend --> flame graph
```

- Metrics: Prometheus/Datadog-like time series systems
- Logs: ELK/OpenSearch/Splunk-like systems
- Traces: Jaeger/Tempo/Datadog APM-like systems

## 7.1.4 Practical example

p99 latency spikes:
1. Dashboard shows **service p99** increased
2. Dependency dashboard shows **DB p99** also increased
3. Traces show `GET /orders` spends most time in DB span
4. Logs for that traceId show query pattern and error hint  
    e.g., missing index after migration → full table scan

So you identify root cause quickly.

## 7.1.5 Real-world use cases
- incident debugging, capacity planning, SLA reporting

## 7.1.6 Interview questions

**Q. What dashboards are mandatory for a new service?**
Mandatory dashboards: **golden signals** (QPS, p50/p95/p99, error codes (4xx/5xx), saturation(CPU, memory, thread pools, DB connections, queue backlog)) + **dependencies** (DB latency/conns, cache hit/latency, downstream RPC, queue lag/DLQ).

 
**Q. How do you avoid alert fatigue?**
By monitoring:
- **Traffic (QPS):** how much load
- **Latency:** p50/p95/p99 response time
- **Errors:** 4xx vs 5xx rate
- **Saturation:** CPU, memory, thread pools, DB connections, queue backlog

**Q. How do you trace requests across microservices?**
We propagate a **traceId/requestId** through headers (`traceparent`/`x-request-id`) across services and include it in logs/spans to correlate end-to-end.

# 7.2 SLO/SLI/SLA + error budgets (how SRE thinking changes design)

## 7.2.1 Intuition
You don’t aim for “perfect.” You aim for **measurable reliability** and spend error budget wisely.

## 7.2.2 Concepts

- **SLI (Service Level Indicator)** = the metric you measure
	Examples:
	- % requests with **latency < 200ms**
	- **availability** (% successful responses)
	- error rate (5xx)
	- durability (lost messages)

- **SLO (Service Level Objective)** = the target for that metric
	Examples:
	- “**99.9%** of requests succeed”
	- “**99.9%** of requests have latency < 200ms”  
	This is what engineering aims to meet internally.
    
- **SLA (Service Level Agreement)** = external promise to customers
	- Usually looser than SLO
	- Has **penalties/credits** if you violate it


**Error budget**
	- allowed failure rate within a window
	- used to decide: ship faster vs stabilize

SLIs are what we measure, SLOs are the targets, SLAs are external promises, and error budgets turn reliability into a lever: if you’re burning budget, you prioritize stability over shipping.

## 7.2.3 Architecture implication
Higher SLO means you must design for fewer failure modes:
- **multi-AZ** deployment
- redundancy (no single point of failure)
- simpler critical dependencies (avoid long chains)
- graceful degradation (serve cached/stale, disable non-critical features)
- better observability and fast rollback

So reliability targets directly change architecture complexity and cost.

**Graceful degradation**
When something breaks, the system still works in a **reduced mode** instead of fully failing.  
Examples:
- show cached/stale profile instead of error
- disable recommendations, still show core page
- queue non-critical work instead of blocking request
 
**Fallbacks**
The **specific alternative path** you take during degradation.  
Examples:
- cache miss + DB slow → serve stale cache / default response
- primary region down → route to secondary region
- downstream service error → skip that call and continue

**Blast radius**
- small blast radius: only 1% users or one feature impacted
- large blast radius: entire service down


## 7.2.4 Practical example
If you burn error budget fast → pause risky deploys, reduce blast radius.

## 7.2.5 Interview questions

**Q. If your SLO is 99.99%, what do you _change_ first?**
For 99.99% SLO, I first design for fewer failure modes: **multi-AZ + redundancy (no SPOF)**, reduce critical dependency chains, add **graceful degradation/fallbacks**, strong **observability + fast rollback**, and **blast-radius control** (canary/feature flags) so failures don’t take down the whole service.


# 7.3 Resilience patterns (timeouts, retries, circuit breakers, bulkheads)

## 7.3.1 Intuition

Most outages are **cascading failures**. Resilience prevents one slow dependency from taking everything down.

Most outages start like this:
- DB becomes slow
- your service threads wait on DB
- thread pool fills up
- requests time out and retry
- traffic doubles/triples
- DB gets even more overloaded
- now multiple services fail → outage spreads

**Resilience patterns** break this chain.

## 7.3.2 Concepts (must-know)
- **Timeouts**: Every network/DB call must have a **time limit**. If dependency is slow, fail fast instead of waiting forever. Timeouts should be layered: DB < service < gateway < client. It prevents thread/connection pool exhaustion.
- **Retries**: only for safe/idempotent operations; backoff + jitter; cap retries
- **Circuit breaker**: Temporarily stop calling a failing dependency and fail fast until it recovers.
- **Bulkheads**: isolate resources (thread pools/connection pools) per dependency
- **Load shedding**: When overloaded, reject/drop non-critical work (429/503) to protect core functionality.
- **Fallbacks**: Serve an alternate response (stale cache/partial/degraded mode) when a dependency is unavailable.

## 7.3.3 Architecture

```
Service  
  |-> Dep A (timeout, retry, breaker, bulkhead)  
  |-> Dep B (timeout, breaker)  
  |-> Cache fallback (stale-ok)
```

For example:
```
Client
  |
  v
Service (your code)
  |
  +--> Dep A client wrapper: timeout + retry + breaker + bulkhead
  |
  +--> Dep B client wrapper: timeout + breaker + (maybe no retry)
  |
  +--> Cache fallback path (stale-ok)
```
 
**Meaning:**
- **Dep A** and **Dep B** are things like DB, Payment service, Search, 3rd-party API.
- You don’t call them “raw”.
- You call them through a **client wrapper** that enforces:
    - **timeout** (don’t hang)
    - **retry** (only if safe)
    - **circuit breaker** (stop hammering when failing)
    - **bulkhead** (dedicated pool so it can’t consume all threads/conns)
Serve an alternate response (stale cache/partial/degraded mode) when a dependency is unavailable.

## 7.3.4 Practical example

When DB slows:
- **Circuit breaker opens** → stop hammering DB
- **Read-only endpoints** serve from cache (maybe stale) so the app stays usable
- **Writes** return 503 with `Retry-After` (don’t pretend success)
- **Background jobs** pause so they don’t compete for DB connections

Net effect: you keep the system partially available, reduce pressure on DB, and recover faster.

## 7.3.5 Interview questions

**Q. When should you NOT retry?**
Don’t retry **non-idempotent/unsafe operations** (most writes) unless protected by idempotency, and don’t retry when the dependency is already unhealthy (timeouts, breaker open, overload) because it amplifies traffic—cap retries and fail fast.
 
**Q. How do you prevent retry storms?**
Prevent retry storms by retrying only **safe/idempotent** ops (writes only with idempotency), using **tight timeouts**, **exponential backoff + jitter**, and a low retry cap; combine with **circuit breakers**/rate limiting to fail fast when a dependency is unhealthy (and for async, poison messages go to DLQ)
 
**Q. Fail open vs fail closed: when?**
- **Fail-open:** choose when **availability > strict protection**, and the system can tolerate risk (e.g., non-critical rate limits, optional features, serving cached reads if a dependency is down).

- **Fail-closed:** choose when **security/correctness/cost > availability** (e.g., auth/permission checks, payments, OTP/login brute-force protection, quota enforcement on expensive endpoints).


# 7.4 Overload management (capacity + backpressure)

## 7.4.1 Intuition

When load exceeds capacity, something must give.

If you don’t control it:
- latency shoots up
- queues build up
- timeouts trigger retries
- system collapses (cascading failure)

So you want **controlled degradation**: reject/slow down work in a predictable way.

## 7.4.2 Concepts

- **Backpressure**: Backpressure means: **don’t accept unlimited work**. Bounded queues, max in-flight requests(e.g., only 200 concurrent requests). When full → return 429/503 or shed non-critical work. This prevents memory blow-ups and thread pool exhaustion.

- **Rate limiting**: At gateway, cap requests per user/tenant/IP, enforces fairness and stops one noisy client from consuming all capacity.

- **Priority queues**: process critical endpoints/premium users first and shed or delay low-priority traffic during overload.

- **Autoscaling**: Autoscaling just means: **add more workers/servers when load increases, remove them when load drops**. It works great only when the thing you’re scaling is the **bottleneck**.

- **Queue lag**: backlog growth is the early warning that producers outpace consumers and SLA risk is rising.

## 7.4.3 Architecture

```
Gateway (rate limit) -> Service (bounded concurrency) -> Queue (buffer) -> Workers
```
- Gateway prevents unfair/abusive load.
- Service limits in-flight work (backpressure).
- Queue buffers excess async work.
- Workers process at a controlled rate; you can scale workers independently.

This structure avoids overload turning into a meltdown.
## 7.4.4 Practical example

Video processing: queue absorbs spikes; worker pool scales; DLQ for failures.

## 7.4.5 Interview questions

**Q. How do you degrade gracefully for a feed service under extreme load?**
We rate-limit at the gateway and apply service-level backpressure; under load we **prioritize premium/critical traffic**, **serve cached/stale feed** and degrade features (drop personalization/recommendations, reduce page size/quality) to cut fanout. Async work goes via queue + workers; autoscale stateless compute only if it’s the bottleneck, otherwise protect the DB with caching/sharding/read replicas and shed non-critical writes.



# 7.5 Security + Privacy (system design essentials)

## 7.5.1 Intuition

Security isn’t just “put JWT”.  
It’s about:
- **preventing abuse** (bots, brute force, scraping)
- **reducing blast radius** (if something is compromised, limit what it can access)
- protecting sensitive data (PII, secrets)

## 7.5.2 Concepts (must-know)

- **AuthN**: who are you? (JWT/OAuth/session)
- **AuthZ**: what can you do? (RBAC/ABAC, resource-level checks). AuthZ often stays inside services because it needs domain data.
- **Least privilege**:Give each service/user only the permissions it needs. Payment service shouldn’t have access to all user data. DB credentials scoped to the right tables/buckets.
- **Secrets management**: Never hardcode secrets in code or logs. Use secret managers / env injection and rotate secrets. Examples: DB passwords, JWT signing keys, API keys.
- **Encryption**:
    - in transit (TLS)
    - at rest (DB encryption)
- **PII handling**:
    - Don’t log PII; **redact** sensitive fields
    - data retention policies: delete old data you don’t need
    - Audit logs: record who accessed sensitive data (especially in SaaS/enterprise)

## 7.5.3 Architecture

```
Client -> Gateway (AuthN, rate limit, WAF)  
           |  
           v  
Service (AuthZ checks, input validation)  
           |  
           v  
Data (encrypted, audited access)
```

- Gateway handles **broad protections**: validate token, rate limiting, WAF.
- Services enforce **real authorization**: resource-level permissions, tenant isolation checks.
- Data layer enforces: encryption + audit + isolation.

This is “defense in depth”: multiple layers, not one gate.

## 7.5.4 Practical example

Multi-tenant SaaS:
You must ensure tenants can’t see each other’s data.
Typical design:
- token includes `tenantId`
- every query is scoped by tenant:
    - row-level: `WHERE tenant_id = ?`
    - or separate schema/db per tenant (strong isolation, higher cost)
- rate limits per tenant (one tenant can’t overload system)

## 7.5.5 Interview questions

**Q. How do you design authz for “user can access only their orders”?**
Verify JWT, extract `userId/tenantId`, and enforce **resource-level authz** by scoping every query/update: `WHERE tenant_id=? AND user_id=?` (and for single order: `AND order_id=?`), returning 404/403 if no match.
 
**Q. How do you handle PII in logs/traces?**
Don’t log PII/secrets—**redact/mask** sensitive fields, correlate using `traceId/requestId` + non-PII IDs (orderId/tenantId), and enforce **retention + access controls + audit** on log access.
 


# 7.6 Deployment + rollouts (ship safely)

## 7.6.1 Intuition

Most production outages are caused by **changes** (code/config/DB). So rollouts are designed to **limit blast radius** by exposing updates to a small slice of traffic first, watching health metrics, and enabling **fast rollback** if anything regresses.

## 7.6.2 Concepts (must-know)

- **Canary**: Route a **small %** of traffic (e.g., 1–10%) to the new version while most users stay on the old one. Monitor errors/latency/business KPIs and gradually ramp up; rollback quickly if metrics regress.

- **Blue/green**: Run two full environments (**blue=current**, **green=new**) and switch traffic over in one move. Rollback is instant by switching back, but it costs more and requires keeping both stacks ready.

- **Feature flags**: Ship code but keep feature **off** until you flip a flag.
	- decouples deploy from release
	- you can enable per user/tenant
	- instant rollback by flipping flag off
	✅ fastest “turn off” control  
	✅ great for gradual rollout

- **Backward compatibility**: Old and new versions run together during rollout, so APIs/events/schema must tolerate both. Use additive changes (don’t remove/rename fields) and make consumers tolerant of unknown fields.

- **Safe DB migrations**: add new schema in a backward-compatible way and deploy code that can handle both old+new. Backfill and switch reads; **Contract:** remove old schema only after old code is fully gone.

## 7.6.3 Architecture

```
LB -> (90%) v1  
   -> (10%) v2  (canary)  
Monitoring decides expand/rollback
```

## 7.6.4 Practical example

Add a new column:
- deploy code that writes both old+new
- backfill
- switch reads
- later drop old

## 7.6.5 Interview questions

**Q. How do you do zero-downtime DB migration?**
We use the **expand/contract** pattern: add the new schema in a backward-compatible way, deploy code that can handle both (often **dual-write**), **backfill** existing data, **switch reads** to the new schema, and only after all old versions are gone do we **drop** the old schema.

**Q. What metrics decide canary success?**
 


# 7.7 Disaster Recovery + Geo design (RPO/RTO, multi-AZ, multi-region)

## 7.7.1 Intuition

DR isn’t “nice engineering”—it’s a **product/business requirement**:
- How much downtime is acceptable?
- How much data loss is acceptable?

Those answers decide whether you need multi-AZ only, or multi-region too.

## 7.7.2 Concepts (must-know)

### RPO (Recovery Point Objective) = “how much data can we lose?”

Example: **RPO = 5 minutes**
- In worst case, you can lose up to the last 5 minutes of writes (because replication/backup lags).

Lower RPO (0–seconds) is harder and more expensive.

### RTO (Recovery Time Objective) = “how long can we be down?”

Example: **RTO = 30 minutes**
- You must restore service within 30 minutes after disaster.

Lower RTO needs automation and hot standby.

### Multi-AZ (within one region)

Deploy across multiple **Availability Zones** (separate datacenters).  
Goal: survive **AZ failure**.

Typical:
- LB spreads traffic across AZ1 + AZ2
- DB has multi-AZ replication/failover

This is baseline for most production systems.


### Multi-region (across regions)

Now you can survive **region failure** (Mumbai region down).  
But it’s much harder because:
- data replication across regions is slower
- latency is higher
- conflict handling becomes hard if both regions accept writes


### Active-passive vs active-active

#### Active-passive
- Region A is serving traffic
- Region B is standby (warm/hot) with replicated data  
    Failover means switching traffic and promoting standby.

✅ simpler, fewer conflicts  
❌ failover takes time, replication lag affects RPO

#### Active-active
- Both regions serve traffic simultaneously
- usually means multi-leader writes

✅ fastest failover + better global latency  
❌ conflicts, ordering, and strong consistency become hard  
Most systems only do active-active for AP-ish workloads (feeds, counters), not money.

## 7.7.3 Architecture patterns

**Multi-AZ (single region)**

```
LB  
 |\  
 | \-> App AZ1 ----\  
 | \-> App AZ2 -----\-> DB (multi-AZ)
```

**Active-passive multi-region**

```
Users -> GeoDNS  
   -> Region A (active) -> DB primary  
   -> Region B (passive) -> DB replica  
Failover: promote replica + switch DNS/LB
```

**Active-active multi-region (hard)**
- multi-leader writes
- conflict resolution
- often limited to “AP-ish” workloads

## 7.7.4 Practical example

Payments:
- usually single-region write for strong consistency
- multi-region reads + DR standby

## 7.7.5 Interview questions

**Q. What RPO/RTO would you pick for a chat app vs payment ledger?**
- **Chat:** I’d target **RPO in seconds (up to ~1 min)** and **RTO in a few minutes**, because user experience favors fast restore and we can tolerate tiny loss with client retries/“undelivered” states.  
- **Payments ledger:** I’d target **RPO ≈ 0** and **RTO minutes (e.g., 5–30)**, because correctness/auditability dominate; I’d prefer single-writer or strongly consistent replication for writes.

**Q. What breaks when you go active-active?**
Active-active makes you a multi-writer system—under lag/partitions you lose simple global ordering/consistency and must handle write conflicts + duplicates via conflict resolution and idempotency.

# Phase 7 Drill (30 minutes)

**Prompt:** “Design a multi-tenant SaaS API platform used globally.”  
Must cover:

- SLOs + dashboards
- resilience patterns (timeouts/retries/breakers)
- security (authN/authZ, tenant isolation, PII)
- safe deployments (canary + migrations)
- DR strategy with RPO/RTO trade-offs