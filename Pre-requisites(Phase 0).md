
# P-0.1: Mental model of a modern distributed system

## Intuition (what you should _feel_)
Any backend system is a **flow of a request** through a few predictable stages.  
Your job in HLD is to decide:
- **Where state lives**
- **How it scales**
- **What happens when things fail**
- **How you observe and recover**

If you can trace a request end-to-end and name the bottlenecks/failures at each hop, you can design most systems.

## Concepts (minimal but complete)

1) Request lifecycle (the core pipeline)
```
User Action
   |
   v
Client (Mobile/Web)
   |
   v
DNS -> (find IP)
   |
   v
TCP/TLS -> (connect securely)
   |
   v
Load Balancer -> (pick healthy server)
   |
   v
App Service -> (business logic)
   |
   +--> Cache (fast reads)
   |
   +--> DB (source of truth)
   |
   +--> Queue/Stream (async work)
   |
   +--> Blob Store (files)
   |
   +--> Search Index (text queries)
   |
   v
Response back to client
```

**2) Stateless vs stateful**
- **Stateless service:** doesn’t store user/session data in memory long-term → easy horizontal scaling.
- **Stateful components:** DB, cache, queues, search, blob store → scaling requires sharding/replication.

Rule of thumb:
- Keep **compute stateless**
- Keep **state in dedicated systems**

**3) Reads vs Writes paths**
### What “path” means
A **path** is the _route a request takes through your system components_.
- **Read path** = when the user wants to **fetch** data
- **Write path** = when the user wants to **change** data

You treat them differently because their priorities are different.

### Read path: optimize for latency

Reads are usually **more frequent** and users feel slowness immediately.

**Typical read flow**
```text
Client -> Service -> Cache/CDN/Search -> (DB fallback) -> Response
```

**Why cache/CDN/indexes help**
- **CDN**: serves static/public responses close to the user (fast)
- **Cache (Redis)**: serves hot data without hitting DB (fast)
- **Search index**: serves text search quickly (DB is slow for search)
- **DB indexes**: make DB queries fast for specific columns

**Read path trade-offs**
- You may return **slightly stale** data (cache TTL, replication lag)
- You must handle **cache miss** and **stampede** (many requests miss at once)

**Example** 
`GET /product/123`
- check Redis → hit → return in 5ms
- miss → DB query 30ms → store in Redis → return


### Write path: optimize for correctness

Writes deal with **money, orders, inventory, permissions** — mistakes are costly.

**Typical write flow**
```text
Client -> Service -> DB transaction (commit)
                    -> queue/event (async side effects)
                    -> response
```

**Key correctness tools**
- **Transactions**: ensure “all-or-nothing”
    - e.g., create order + reserve inventory together
- **Idempotency**: retry-safe write 
    - if client retries POST, you don’t create duplicates
    - often done with an `Idempotency-Key`
- **Ordering**: events happen in correct sequence
    - e.g., “OrderPaid” should not be processed before “OrderCreated”

**Write path trade-offs**
- Usually slower than reads
- Often uses async processing (queue) to keep API responsive

**Example**  
`POST /orders`

- DB transaction creates order `PENDING`
- publish `OrderCreated` event
- worker later charges payment, updates status

**4) Critical distributed realities**
### A) Networks fail

**What happens**
- packets drop, latency spikes, partial partitions
- request hangs or takes forever

**What you do**
- **timeouts**: don’t wait forever
- **retries with backoff**: retry safe operations carefully
- **circuit breaker**: stop calling a dead dependency

### B) Machines fail

**What happens**
- servers crash/restart
- in-memory data disappears
- disks can corrupt / fill up

**What you do**
- **replicas**: multiple instances behind LB
- keep **services stateless**
- store state in **durable systems** (DB, Redis, S3)

### C) Dependencies fail (DB/cache/queue)

**What happens**
- DB slow → app threads block → timeouts cascade
- cache evicts keys → sudden DB load spike
- queue backlog → delayed processing, SLAs missed(Service Level Agreements)
 **SLA** = Promise a system/service makes about **performance, availability, or response times**.

**What you do**
- **fallbacks**:
    - reads: serve stale cache / degrade features
    - writes: accept request, queue it, respond “processing”
- **backpressure**: Limit how much work your app sends to a slow DB or queue → prevents cascading failures.
- **autoscale workers**: If queue grows, spin up more workers to process tasks faster.
- **DLQ(Dead Letter Queue):** for poison messages

### D) Time is unreliable

**What happens**
- clocks drift across machines
- events arrive out of order
- “latest” based on timestamps becomes wrong

**What you do**
- don’t depend on timestamps for ordering across nodes
- use **version numbers** / sequence per entity
- use partition ordering in streams where needed


### So designs must include: timeouts + retries + fallback + monitoring

#### Timeouts
Stop “hanging” calls so one slow dependency doesn’t kill the whole service.

#### Retries
Handle transient failures — but only with:
- backoff + jitter
- max retry limit
- idempotency (especially for writes)

#### Fallback
When something is down, return something acceptable:
- stale cache, partial response, “try later”, async processing

#### Monitoring
You can’t run distributed systems without observability:
- metrics: latency/error rate/saturation
- logs: structured with requestId
- traces: see which hop is slow
- alerts: DB p99, queue lag, cache hit rate


## Architecture (baseline you’ll reuse everywhere)

**Baseline architecture (single region, scalable stateless compute):**

```sql
          +-------------------+
Users --> | DNS (name -> IP)  |
          +-------------------+
                    |
                    v
          +-------------------+
          | Load Balancer     |
          | (health checks)   |
          +-------------------+
                    |
        +-----------+-----------+
        |                       |
        v                       v
+---------------+       +---------------+
| App Server A  |       | App Server B  |   (stateless)
+---------------+       +---------------+
        |                       |
        +----------+------------+
                   |
                   v
        +----------------------+
        | Cache (Redis)        |  (optional)
        +----------------------+
                   |
                   v
        +----------------------+
        | DB (primary/replica) |
        +----------------------+
                   |
                   v
        +----------------------+
        | Observability        |
        | logs/metrics/traces  |
        +----------------------+

```

**Where the other blocks fit**
- **Queue/Stream** for async tasks (emails, thumbnails, payouts)
- **Blob store** for images/videos/docs
- **Search** for text queries
- **CDN** for global low-latency static + media

## Practical example (trace one request)
Example: `POST /orders` (place an order)
1. Client sends request
2. DNS resolves `api.company.com`
3. TLS handshake → encrypted channel
4. LB routes to a healthy app server
5. App server:
    - validates request
    - checks inventory/pricing
    - writes order in DB (source of truth)
    - publishes event to queue (“OrderPlaced”)
6. Response returns `201 Created` with `orderId`
7. Async workers consume event → send email, analytics, fulfillment, etc.

## Real-world use cases (why this mental model matters)
- You’re debugging latency: **is it DNS/TLS/LB/app/DB?**
- You’re scaling: **stateless compute scales easily; state needs strategy**
- You’re adding a feature: decide **sync vs async**, **cache vs DB**, **index vs scan**
- You’re improving reliability: plan **timeouts/retries/circuit breakers** and dashboards

## Interview-level practice questions (for P0.1)

**Q. In the baseline pipeline, where do you typically add caching, and why?**
Typically cache sits between service and DB (Redis) for hot reads and to protect DB. For static/public content I also use CDN. I’d choose cache-aside with TTL + stampede protection.

 
**Q. What parts can you scale horizontally with almost no risk, and what parts cannot?**
**Easy to scale horizontally (low risk):**
- **Stateless app servers**
- **Queue consumers/workers** (usually)
- **Read replicas** (for read-heavy DB)
- **CDN/edge**
Stateless compute scales horizontally easily. Stateful systems (DB writes, cache correctness, streams ordering, search reindexing) need sharding/replication and careful consistency/operability.


**Q. If DB latency spikes from 20ms to 500ms, what happens to the system end-to-end?**
DB becomes the bottleneck → app threads + DB connection pool saturate, p99 latency and timeouts spike, and retries can amplify into a retry-storm causing cascading failures; you need timeouts/circuit breakers + load shedding/cache fallback to protect the DB.
 
**Q. When would you introduce a queue instead of making a call synchronous?**
When the work is slow or involves retries/DLQ, unreliable downstreams or spike buffering - so that the API stays fast and resilient while processing happens asynchronously with idempotency
 
**Q. What are 5 common failure points in this pipeline and one mitigation for each?**
1. LB/Gateway overload → rate limiting + load shedding.
2. App instance crash → multi-replica + health checks/graceful shutdown.
3. DB slow/unavailable → timeouts + circuit breaker (plus read replicas/index tuning).
4. Cache miss/eviction/stampede → TTL + singleflight/locking + DB fallback.
5. Queue backlog/poison messages → autoscale consumers + backpressure + DLQ.

## Extra Notes
### 1️⃣ **Cache-Aside Pattern**
Also called **Lazy Loading**.
**Meaning:**
- Your application **does not write to cache automatically**.
- Instead, on a request:
    1. Check cache first.
    2. If cache **hit → return value**
    3. If cache **miss → fetch from database**, then **populate cache**

**Use case:**
- Expensive DB queries or slow API calls
- You want cache to reduce load

### 2️⃣ **TTL (Time-To-Live)**

- TTL is **how long a cached entry remains valid**.
- After TTL expires → cache entry is considered stale → next request reloads it from DB

Example:
`Key: user:123 Value: {name: "Alice", age: 25} TTL: 5 minutes`
After 5 minutes → cache miss → reload from DB

### 3️⃣ **Stampede Protection**

Also called **Cache Stampede / Dogpile protection**
**Problem:**
- Many requests hit the cache **at the same time** after TTL expires
- All of them go to the database → DB overloaded

**Solution / Protection:**
- Techniques to **serialize or randomize cache reloads**:
    - **Locking / Mutex**: Only 1 request reloads the DB, others wait
    - **Early refresh / probabilistic TTL**: Refresh cache a bit before it expires
    - **Request coalescing**: Merge duplicate DB queries


# P0.2 HTTP essentials (what you must know for HLD)

## Intuition (what you should _feel_)
HTTP is not “just how requests are sent.” In system design, HTTP is:
- a **contract** (what the API promises)
- a **failure language** (how errors are communicated)
- a **reliability tool** (idempotency, retries, caching)
- a **performance lever** (keep-alive, caching headers, compression)

If your HTTP/API semantics are sloppy, your system becomes flaky under retries, load, and partial failures.

## Concepts (only what matters in interviews)

### 1) Methods: semantics matter
- **GET**: read. **Safe** + **idempotent** (should not change state).
- **POST**: create / action. Not idempotent by default.
- **PUT**: replace resource (full update). **Idempotent**.
- **PATCH**: partial update (only fields). Often idempotent _if designed carefully_.
- **DELETE**: delete resource. Typically idempotent.

**Interview rule:** choose methods to match _meaning_, not convenience.

### 2) Status codes: your system’s “truth table”

You should be fluent with these:
**Success**
- **200 OK**: response body present (GET/PUT/PATCH).
- **201 Created**: new resource created (POST) + return ID (and maybe Location header).
- **204 No Content**: success but no body (DELETE).

**Client errors**
- **400 Bad Request**: malformed/invalid input (schema/validation).
- **401 Unauthorized**: missing/invalid authentication (token missing/expired).
- **403 Forbidden**: authenticated, but not allowed (RBAC).
- **404 Not Found**: resource doesn’t exist (or hide existence for security).
- **409 Conflict**: state conflict (duplicate unique key, version mismatch).
- **422 Unprocessable Entity**: syntactically valid but semantically invalid (optional; many teams use 400).
- **429 Too Many Requests**: rate limited.

**Server / dependency errors**
- **500 Internal Server Error**: unhandled server issue.
- **502 Bad Gateway**: proxy/LB got invalid response from upstream.
- **503 Service Unavailable**: overloaded/maintenance; retry later.
- **504 Gateway Timeout**: proxy timed out waiting upstream.


**Golden interview line:**
- Use **409** for “your request is valid but clashes with current state.”
- Use **503/429** for “system is protecting itself.”

### 3) Headers you must know (and why)

#### Request headers

- **Authorization: Bearer `<Token>` → auth**
-  **Content-Type: application/json** → how body is encoded
- **Accept: application/json** → what client expects
- **Idempotency-Key**: `<uuid>` → safe retries for POST (payments/orders)
- **X-Request-Id / Traceparent** → correlation across services

#### Response headers

- **Cache-Control / ETag / Last-Modified** → caching + conditional GET
- **Retry-After** → tells client when to retry (429/503)
- **Location** → where created resource lives (201)


### 4) Idempotency + retries (this is huge in HLD)

**Why it exists:** networks fail → clients retry → your server may receive the same request multiple times.

#### Idempotent means:
Same request processed multiple times ⇒ **same final effect**.
**Examples**
- `GET /users/1` idempotent.
- `PUT /users/1 {name:"A"}` idempotent.
- `POST /orders` **not** idempotent by default (could create duplicates).

### How to make POST safe

Use an **Idempotency-Key** (or clientRequestId) + dedupe store.

**Flow**
```
Client ----POST /orders (Idempotency-Key=K1)----> Service
Service:
  if K1 exists -> return saved response
  else:
     create order in DB
     store (K1 -> orderId + response) with TTL
     return 201
```

**Where stored?**
- Redis (fast) + DB (durable) OR DB only (simpler, slower)
- TTL depends on retry window (minutes/hours)


### 5) Pagination (must for feeds/search)

Two common models:
#### Offset pagination
`GET /items?limit=20&offset=40`
- ✅ simple
- ❌ slow for large offsets, inconsistent if data changes

#### Cursor pagination (preferred)
`GET /items?limit=20&cursor=eyJ0cyI6...`
- ✅ stable + scalable
- ✅ good for infinite scroll
- ❌ needs careful cursor encoding

**Interview default:** cursor pagination for feeds/timelines.


### 6) Versioning (don’t break clients)

- Path: `/v1/orders`
- Header: `Accept: application/vnd.company.v1+json`
- Query: `?version=1` (less preferred)

**Rule:** design for backward compatibility; deploy new fields as optional.


### 7) Caching semantics (HLD-level)
- **Cache-Control: max-age=...** → allow caching
- **ETag** + `If-None-Match` → conditional GET (save bandwidth)
- Avoid caching personalized responses unless carefully scoped.


#### Architecture patterns (how this shows up in real systems)

### A) Standard API Gateway pattern

```
Client
  |
  v
API Gateway (auth, rate-limit, routing)
  |
  v
Service (business logic)
  |
  +--> DB/Cache/Queue
```

### B) Consistent error model (interview-friendly)

Return JSON errors:
- `code` (stable), `message` (human), `details` (optional), `requestId`


#### Practical example (design 2 endpoints)

### 1) Create order (idempotent)

- `POST /v1/orders`
- Headers: `Idempotency-Key`, `Authorization`
- Response: `201` + `{ orderId, status }`

### 2) List orders (cursor pagination)

- `GET /v1/users/{id}/orders?limit=20&cursor=...`
- Response: `200` + `{ items: [...], nextCursor }`


#### Real-world use cases

- Payments: idempotency is mandatory (no double charge)
- Feeds: cursor pagination prevents duplicates/missing items
- High traffic APIs: caching + 304 saves cost
- Microservices: correlation headers make debugging possible


#### Interview practice questions (P0.2)

**Q. For a “create payment” API, how do you prevent double charge on retries?**
For every `POST /create-payment`, I’ll require an **Idempotency-Key** header and **persist it with a unique constraint** mapped to the created payment intent/charge—so on retries/multiple clicks we **return the same result** instead of charging again (and we pass the same idempotency key to the payment provider if supported).

 
**Q. When would you return 409 vs 400? Give examples.**
- **409**: conflict with current resource state (optimistic locking/version mismatch, unique constraint) ✅
- **400**: syntactically/semantically invalid request payload ✅


**Q. Design pagination for a timeline: offset vs cursor—what trade-offs?**
Offset is simple and supports page jumps, but it’s slow for deep pages and causes duplicates/misses on timelines due to inserts shifting offsets. Cursor pagination (using a stable cursor like `(createdAt,id)`) is fast and consistent for infinite scroll, but harder to implement and doesn’t support random page jumps or total pages easily.
 
**Q. What status code would you use when your service is overloaded and why?**
I’d use **503 Service Unavailable** when the service is temporarily unable to handle requests due to overload (often with a **Retry-After**), and **429 Too Many Requests** when the overload is caused by a specific client exceeding rate limits.
 
**Q. What headers do you add to trace a request across microservices?**
X-Request-Id / Traceparent** → correlation across services


# P0.3 Networking basics (only what matters in HLD)

## Intuition (what you should _feel_)

In distributed systems, “the network” is the **default bottleneck and failure source**.  
Most real outages are some combo of:
- latency spikes (tail latency)
- packet loss / partial failures
- bad timeouts + retries causing traffic storms
- DNS / TLS / LB misconfig

If you can reason about **where latency comes from** and **how failures propagate**, you can design resilient systems.


## 1) The request path (with where time is spent)

```
Client
  |
  | 1) DNS lookup (maybe cached)
  v
DNS Resolver  -> returns IP
  |
  | 2) TCP connect (handshake)
  v
Server/LB
  |
  | 3) TLS handshake (if HTTPS)
  v
Secure Channel
  |
  | 4) HTTP request/response (bytes over wire)
  v
App / Upstream services
```

### Latency contributors you should name in interviews
- **DNS**: first lookup cost; TTL caching reduces it
- **TCP handshake**: extra RTT
- **TLS handshake**: extra RTT + crypto (often biggest “first request” cost)
- **Network RTT**: physical distance (India ↔ US is slower than India ↔ Mumbai)
- **Serialization**: JSON/Proto size
- **Queueing**: LB/server under load (tail latency)
- **RTT** means **Round-Trip Time** - It’s the **total time for a message to go from client → server and for the reply to come back** (one full “there and back” trip), usually measured in **milliseconds**.


## 2) DNS basics (HLD version)

**What DNS does:** Name → IP resolution with caching.
**Key term: TTL**
- TTL controls how long resolvers cache answers.
- **Low TTL**: faster failover/traffic shifting, more DNS traffic
- **High TTL**: cheaper/faster steady state, slower failover
- TTL means **how long a cached DNS answer is allowed to live** before it must be considered **expired** and re-fetched.

**Where DNS sits in design decisions**
- Multi-region routing (GeoDNS)
- Failover strategy (primary region down)
- Blue/green via DNS (slow unless TTL low)



## 3) TCP vs UDP (what you actually need)

**TCP is “connect first, then talk”**.  
**UDP is “talk without connecting”**.

- **TCP**: reliable, ordered stream (HTTP/1.1, HTTP/2 runs over TCP)
- **UDP**: no guaranteed delivery/order; used for latency-sensitive protocols (QUIC/HTTP/3 uses UDP) 

**Interview takeaway:** most backend APIs assume TCP reliability, but you still handle **timeouts and retries**.

DNS itself usually uses UDP
```
Client  --UDP-->  DNS Resolver (port 53)
          "api.company.com?"
Client  <--UDP--  "Here’s the IP/LB hostname (+ TTL)"
```

Why UDP for DNS?
- It’s small, fast, and one-question/one-answer fits well.

**UDP case exists too (modern web):**
**HTTP/3 runs over QUIC, and QUIC runs over UDP**
```
Client --UDP--> Server (port 443)
   (QUIC “connection” setup happens at QUIC layer, not TCP)
```

So: UDP doesn’t do a TCP handshake, but **QUIC builds reliability + encryption + streams on top of UDP**.

## 4) TLS (HTTPS) essentials

**TLS gives:**
- encryption (privacy)
- integrity (no tampering)
- authenticity (certs)

**Why it matters in HLD**
- adds handshake latency (first request)
- termination point matters:
    - terminate at **LB/API Gateway** (common)
    - or end-to-end TLS to service mesh (advanced)

**Typical pattern**

```
Client --TLS--> Load Balancer/API Gateway --(internal TLS optional)--> Service
```


## 5) HTTP connection reuse (keep-alive) and why “first request is slow”

If you don’t reuse connections:
- every request pays **DNS + TCP + TLS** again → huge latency
    

With keep-alive / pooling:
- pay handshake once, reuse for many requests

**Diagram**

```
Without reuse:
Req1: DNS+TCP+TLS+HTTP
Req2: DNS+TCP+TLS+HTTP  (bad)

With reuse:
Req1: DNS+TCP+TLS + HTTP
Req2:            HTTP  (good)
Req3:            HTTP
```

## 6) Load balancers and health checks (network view)

**LB role:** route traffic to healthy instances and absorb connection complexity.

**Important networking behaviors**
- **Health checks**: remove bad instances
- **Connection draining**: allow in-flight requests to finish on deploy
- **Sticky sessions**: tie client to one server (avoid unless needed)
- **L4 vs L7**:
    - L4: TCP-level (fast, less aware)
    - L7: HTTP-level (routing by path/host, headers, can do auth, rate-limit)


## 7) Timeouts, retries, and retry storms (most important part)

**Golden rule:** retries amplify traffic.
If a downstream gets slow:
- upstream times out and retries
- now downstream gets **2x/3x** traffic while already slow
- system collapses (thundering herd)

**Quick Diagram**
```
Client  ->  Gateway  ->  Service  ->  DB
  ^          ^           ^           ^
upstream   upstream    upstream    downstream
```

More precisely, for each hop:
- **Gateway is upstream of Service**
- **Service is downstream of Gateway**
- **Service is upstream of DB**
- **DB is downstream of Service**

### Correct pattern: Timeout + retry with backoff + jitter
- Timeouts stop threads/connections from being stuck forever.
- **Timeouts** must be smaller than the caller’s timeout budget
- use **exponential backoff**
	Instead of retrying immediately, wait longer each time:
	50ms → 100ms → 200ms → 400ms (example)

- add **jitter** (randomness) so everyone doesn’t retry together
	If everyone retries at exactly 100ms, they all hit together (thundering herd).  
	Add randomness:
	retry at 80–140ms instead of exactly 100ms.

- cap retries(retries must be limited); prefer **fail fast** for non-critical paths
- Also: only retry **safe operations** (idempotent). For writes, require **idempotency keys**.

**Budget diagram**

```
Client timeout: 2s
  Gateway timeout: 1.8s
    Service timeout: 1.5s
      DB timeout: 200ms
```

If inner timeouts are longer than outer, you get wasted work + cascading failure.

### Why inner must be smaller than outer
If DB timeout were 1.6s but service timeout is 1.5s:
- service gives up at 1.5s, but DB keeps working until 1.6s
- wasted DB work + connections held → pool exhaustion → cascading failure

So “budgeting” prevents **wasted work** and **resource saturation**.

* Circuit breaker - If a dependency is failing/slow, stop calling it temporarily:
	- protects downstream
	- prevents retry storms
	- enables fast fallback
- Backpressure / load shedding- When saturated, you must reject some work:
	- return 503/429
	- drop non-critical tasks
	- queue writes (if safe) instead of blocking


## 8) Partial failures (the real “distributed” problem)

Not all failures are total.
- one AZ(Availability Zone) is degraded. One specific AWS Availability Zone is **still “up”**, but it’s **performing badly** or **partially failing** compared to other zones. “AZ degraded” = the zone is alive but unreliable/slow, so some portion of your traffic suffers.
- **Some packets drop**: network is flaky → intermittent timeouts.
- **One DB shard slow**: only users mapped to that shard see high latency.
- 1% requests fail → but retries can turn it into 20%

**Mitigations**
- circuit breakers
	**Idea:** when a dependency is failing/slow, **stop calling it for a short time**.
	- Prevents wasting resources on requests likely to fail
	- Protects downstream from more load
	- Allows fast fallback (cached data, default response)

	Think: “Dependency is sick → don’t keep poking it.”


- load shedding
	**Idea:** when your service is overloaded, **reject some traffic early** (503/429) instead of letting everything queue up and time out.
	- Keeps p99 sane for the requests you do accept
	- Prevents cascade (thread/connection pool exhaustion)
	Think: “Better to fail 5% fast than fail 80% slowly.”


- bulkheads (isolate pools)
	**Idea:** isolate resources so one bad dependency doesn’t take down everything.  
	Examples:
	- Separate thread pools: `dbPool`, `paymentPool`, `searchPool`
	- Separate connection pools per downstream
	- Separate queue consumers per topic/priority
	So if “Search” is slow, it doesn’t consume all threads and block “Payments”.


- hedged requests (advanced, used carefully)
	**Idea:** if a request is taking unusually long, send a **second request** to another replica and take the first successful response.

	Example:
	- Call replica A
	- If not done by p95 threshold (say 150ms), also call replica B
	- Use whichever returns first, cancel the other if possible
**Why it helps:** reduces tail latency when “some replicas are slow”.

**Why it’s dangerous:** it increases traffic, so you must:
- hedge only on the slowest tail    
- cap concurrency
- use only for safe/idempotent operations (mostly reads)

## 9) What you should say in interviews (network-aware)

When presenting an architecture, call out:
- expected RTT (regional vs cross-region)
- timeout and retry policy per hop
- LB health checks and rollout behavior
- how you’ll avoid retry storms
- observability: latency percentiles (P50/P95/P99)


## Practical example (network reasoning)

You have users in India, servers in US-East. Complaints: “app feels slow”.  
Network reasoning:

- RTT is high + TLS handshake cost  
    Fix options:
	- move to India region / multi-region
	- use CDN for static + edge caching
	- keep-alive/connection pooling
	- compress responses, reduce payload


## Real-world use cases
- Multi-region deployments and failover
- Mobile apps with flaky networks
- Microservices where one dependency slows down
- Zero-downtime deploys (connection draining + readiness)


## Interview practice questions (P0.3)

**Q. Why is the first request after app start often slower?**
First request is often slower due to one-time setup costs: **DNS lookup + TCP/TLS handshake**, plus **cold start/warm-up** on the server (JVM/JIT/classloading, lazy init) and **warming connection pools** to DB/Redis/HTTP clients; after that, keep-alive/pools reuse connections so you mostly pay the normal HTTP + processing cost.
 
**Q. How do timeouts need to be set across gateway → service → DB?**
Set **timeout budgets** so each downstream call has a smaller deadline than its caller (DB < service < gateway < client) to avoid wasted work and cascading pool exhaustion—e.g., client 2s, gateway 1.8s, service 1.5s, DB 200ms. Retries (if any) use **backoff + jitter** within that budget.
 
**Q. What causes a retry storm, and how do you prevent it?**
A retry storm happens when a **downstream becomes slow/partially failing**, upstream requests **timeout and automatically retry**, and those retries **synchronize** (thundering herd), amplifying traffic and overwhelming the dependency. To prevent it: set **strict timeout budgets per hop**, retry only **idempotent** operations with **exponential backoff + jitter** and **low caps**, add **circuit breakers/bulkheads** and **load shedding (503/429)**; for critical writes use **idempotency keys** so retries don’t create duplicates.

**Q. When would you use L7 vs L4 load balancing?**
“**L4** load balancing works at the TCP/UDP level (no HTTP awareness) — it’s faster/simpler and good for generic TCP services. **L7** works at the HTTP layer, so it can do host/path/header-based routing, TLS termination, rate limiting/auth, and canary/blue-green, but adds more overhead and complexity
 
**Q. What’s the trade-off of low vs high DNS TTL?**
Higher TTL brings the issue of slow failover whereas lower TTL comes brings the issue of higher DNS traffic.


# P0.4 Data fundamentals (keys, indexes, transactions) — HLD level

## Intuition
Databases don’t store “tables”; they store **data + access paths**.  
In HLD, you win by designing around:
- **primary access patterns** (what queries dominate)
- **correctness boundaries** (what must be transactional)
- **scaling method** (replication vs partitioning)

## Normalization
**Normalization** is a database design approach to **reduce data duplication** and **avoid inconsistencies** by splitting data into well-structured tables and linking them with keys.

### Types (normal forms)

- **1NF (First Normal Form):** Keep values **atomic** (no lists/arrays in a single column).  
    Example: don’t store `"item1,item2"` in one field—use an `order_items` table.

- **2NF (Second Normal Form):** If a table has a **composite key**, every non-key column must depend on the **whole key**, not part of it.  
    Example: in `(orderId, productId)` table, product name depends only on `productId` → move to `products`.

- **3NF (Third Normal Form):** Non-key columns should not depend on other non-key columns (**no transitive dependency**).  
    Example: `deptName` depends on `deptId`, not on `empId` → move department data to a `departments` table.

- **BCNF (Bonus):** Stronger version of 3NF; every determinant must be a candidate key (helps in edge cases with overlapping keys).

## Concepts
**1) Keys**
- **Primary key (PK):** unique identifier (fast point lookups).
- **Natural vs surrogate keys:** choose surrogate (UUID/long) when natural keys change.
- **Uniqueness constraints:** enforce business invariants (email unique, orderId unique).

**2) Indexes (what they buy + what they cost)**
- **What they buy:** fast lookups/filtering/sorting without scanning full table.
- **What they cost:** slower writes + more storage + maintenance overhead.

Rule: create indexes for your **top read queries**, not “just in case”.

**3) ACID (why it matters)**
- **Atomicity:** all-or-nothing write set
- **Consistency:** invariants preserved (constraints)
- **Isolation:** concurrent transactions don’t corrupt results
- **Durability:** committed data survives crashes

**4) Transaction boundary**  
Put a transaction around operations that must be **all-or-nothing**.  
Example: create `order` + reserve inventory + create payment intent (often not all in one DB → use saga later).

**5) Isolation (only what matters)**
- Concurrency anomalies exist: dirty reads, lost updates, write skew.
- HLD takeaway: if multiple writers update same row/key, you need **locking/versioning**.
- Instead of “all-or-nothing” in one DB transaction, a saga does:
- **A sequence of local transactions** (each service commits in its own DB)
- If any step fails, it runs **compensating actions** to undo earlier steps

**6) Modeling for access patterns**  
Start from queries:
- “Get orders by userId sorted by createdAt” → index `(userId, createdAt)`
- “Get latest messages in conversation” → partition by `conversationId`, sort by `messageSeq`


## Architecture mental model

App -> DB  
     -> (Index enables fast query)  
     -> (Constraints enforce invariants)  
     -> (Transactions enforce atomic changes)

## Practical example  
E-commerce:
- PK: `order_id`
- Query: list orders for user → index `(user_id, created_at desc)`
- Transaction: create order + order_items together

## Real-world use cases
- Designing schemas, avoiding slow queries, keeping writes correct under concurrency.

## Interview questions

**Q. What query would cause a full table scan here? How would you fix it?**
A full table scan happens when the query **can’t use an index**—e.g., filtering/sorting on non-indexed columns, using functions like `LOWER(email)` or patterns like `LIKE '%x'`, or deep `OFFSET` pagination. Fix it by adding the right **(composite/functional/partial) indexes**, rewriting predicates to be index-friendly, and using **cursor pagination** instead of large offsets.

**Q. When would you denormalize? What breaks when you do?**
We denormalize when the system is **read-heavy** and we need **faster queries** (fewer joins / precomputed views). Trade-offs: **duplicate data** can become **inconsistent or stale**, writes become more complex (update multiple places), and storage/maintenance cost increases.

**Q. Why can “too many indexes” hurt?**
Too many writes can cause slower write operation, because every INSERT, UPDATE, DELETE must update each index. It also adds extra disk memory and requires maintenance overhead.



# P0.5 Concurrency basics (HLD-level)

## Intuition
Concurrency is “many things happen at once.” Distributed systems are concurrency **across machines**.  
Most production bugs are:
- **duplicate processing** (same event handled twice)
- **reordering** (events arrive out of order)
- **races** (two actors update same state)
- **amplification** (retries increase load)


## Concepts

### Race condition

A **race** = outcome depends on timing.
**Example: overselling inventory**  
Two users buy the last item at the same time:
- Request A reads stock = 1
- Request B reads stock = 1
- Both decrement → stock becomes -1 → you sold 2 items ❌

**Fix patterns**
- lock row (`SELECT FOR UPDATE`)
- or optimistic version check (`UPDATE ... WHERE stock>=1 AND version=x`)

### Idempotency (must-have)

**Idempotency** means: _processing the same request multiple times leads to the same final result._
Why you need it:
- networks fail → client retries
- queues are at-least-once → duplicate deliveries happen

**Example: create payment**  
If the client retries POST, you must not charge twice → use `Idempotency-Key` + dedupe store..

**3) Exactly-once vs exactly-once effects**
- “Exactly once delivery” is hard.
- Most systems do **at-least-once** and guarantee **exactly-once effects** via dedupe/idempotency.

**4) Concurrency control patterns**
- **Optimistic(no lock unless conflict):** version number / compare-and-swap (`UPDATE ... WHERE version = x`). Best when conflicts are rare.
- **Pessimistic(lock now):** locks / “SELECT FOR UPDATE”. best when conflicts are common / correctness critical, but reduces parallelism. Lock the row before changing.
- **Single-writer(one lane):** funnel updates through one leader/partition. avoids races by design (used in Kafka partitioning, actor model).

**5) Async vs sync**
- Sync = User waits for everything to finish.  
	✅ simpler consistency  
	❌ higher latency, more coupling, harder during downstream slowness

- Async = Put work in queue/stream, worker processes later.  
	✅ resilient, smooth spikes, retries/DLQ possible  
	❌ eventual consistency, dedupe needed, harder debugging


## Architecture mental model
```
Clients
  | (retries happen)
  v
Service -----> DB (use versioning/locks)
   \
    \-----> Queue (at-least-once) -> Worker (dedupe)
```

## Practical example

Payment webhook arrives twice:
- store `event_id` in a dedupe table
- if seen, ignore; else apply effect

## Real-world use cases
- messaging, payments, inventory, schedulers, webhook consumers.

## Interview questions

**Q. How do you prevent double-processing when a consumer restarts?**
We include a stable **eventId/idempotency key** with each message and keep a **durable dedupe store** (unique constraint) so replays after restart are ignored; the handler also does **state-machine checks** (already processed → no-op) and we **ack/commit the message only after the side effect is committed**.
 
**Q. “At least once” delivery: what code patterns make it safe?**
Message delivered only once is very hard across networks. The use of dedupe store along with the idempotency key where server runs the state machine runs with deduped table keyed with idempotency key or an event id.
 
**Q. When do you prefer optimistic locking vs locks?**
 I'd prefer optimistic locking when the conflicts are rare and I'd prefer pessimistic locking when the conflicts are common, and will use single writer when I want only one leader or consumer to update the respective data to avoid race conditions. 


# P0.6 Linux + debugging basics (production survival kit)

## Intuition
Great HLD includes: **how you operate it**.  
If your design can’t be debugged quickly, it will fail in real life.

## Concepts
**1) Health endpoints**
- **Liveness:** is process alive? (restart if false)
- **Readiness:** can it serve traffic? (remove from LB if false)

**2) Logs**
- Structured logs (JSON) beat plain strings
- Always include:
    - `timestamp`
    - `level`
    - `requestId/traceId`
    - `userId/tenantId` (if safe)
    - key event fields (orderId, convId)

**3) Metrics (minimum set)**
Metrics answer: **“How bad is it?” “Is it getting worse?”**
Minimum set:
- **QPS** (traffic): **QPS** means **Queries Per Second** — basically **how many requests your service receives (or processes) per second**. **Peak QPS**: highest bursts.
- **error rate** (5xx, 4xx)
- **latency percentiles** (P50/P95/P99)
    - P99 is “tail latency” — what users complain about.
    - P99 means 99 percent of requests gets completed in x time period.

Saturation metrics:
- CPU, memory
- DB connections (pool usage)
- queue lag/backlog

**Why:** If P99 spikes, metrics tell if it’s CPU-bound, DB-bound, or queue backlog.

**4) Tracing(where time is spent across microservices)**

Tracing links calls across services in one timeline.
Example:  
`Gateway -> OrderSvc -> PaymentSvc -> DB`
Trace shows:
- OrderSvc spent 900ms waiting on DB
- PaymentSvc took 50ms  
    So root cause is obvious.

**Why:** In microservices, logs alone don’t show cross-service latency well.

**5) Minimal “what’s wrong?” playbook**
- **Is service reachable?** (network/LB/port)
- **Is it healthy?** (readiness failing? instances removed?)
- **Is DB slow?** (DB latency + connection pool full?)
- **Are errors spiking?** (4xx vs 5xx tells client vs server problem)
- **Is queue lag growing?** (consumers stuck? downstream slow?)

This avoids random guessing.

## Architecture mental model
```
Service
  |-- logs -> log store
  |-- metrics -> time series DB -> alerts
  |-- traces -> trace backend
  |-- /health + /ready
```

## Practical example  
Incident: P99 latency up.
- Check metrics: is DB latency up or app CPU?
- Check traces: where time spent?
- Check logs with requestId: error patterns?

## Real-world use cases
On-call, scaling, deployments, RCA writing.

## Interview questions

**Q. What dashboards do you create for a new service on day 1?**
- **RED/Golden signals:** QPS (traffic), error rate (4xx/5xx), latency p50/p95/p99, saturation (CPU/mem/threadpool).
- **Dependencies:** DB latency + connection pool usage, cache hit/miss + latency, downstream call latency/error.
- **Queue (if any):** lag/backlog, consumer throughput, DLQ rate.
- **Infra:** instance health/readiness, restarts, GC (if JVM), node/container CPU/mem.
 
**Q. How do you differentiate client errors vs server errors operationally?**
When we get errors like 4xx it is usually client side errors, like a bad request, or unauthorized or forbidden or too many requests, and in case of server side error it is usually 5xx 

**Q. What does “queue lag” indicate?**
 **Queue lag** means the queue is **building up** because **messages are coming in faster than workers can process them**. It usually indicates **slow consumers, too few consumers, or a slow downstream dependency**, and it leads to **increasing processing delay / SLA risk**.


# P0.7 Cloud basics (conceptual only, interview useful)

## Intuition
Cloud is just **managed building blocks**. In interviews, you should speak in **primitives** first, then map to cloud services if asked.

## Concepts
**1) Regions & AZs**
- **Region:** geographic area (Mumbai, Singapore)
- **AZ(Availability Zone):** isolated datacenter(s) inside region
- Multi-AZ improves availability for local failures.

**2) Networking boundary**
- VPC/VNet: private network for your services
- Public entry: LB/API gateway
- Private subnets for DBs/internal services

- **VPC/VNet:** _The entire office building._
- **Subnet:** A specific floor/wing inside the building.
- **Public subnet:** _The lobby area that has a public entrance from the street.
- **Private subnet:** _Restricted floors accessible only from inside the building (no direct entry from outside)._

**3) Core managed primitives**
- Compute: VMs/containers/serverless
- Storage: object store (blob), block, file
- DB: managed relational/noSQL
- Messaging: queue/pubsub/stream
- CDN: edge caching
- Observability: logs/metrics/traces

**4) Cost and scaling knobs**
- Scale out stateless compute (easy)
- DB scales vertically first, then via read replicas/sharding
- Egress bandwidth costs matter (CDN reduces)

**5) Reliability knobs**
- Multi-AZ: survive AZ outage
- Multi-region: survive region outage (more complexity)


## Architecture mental model
```
Users -> Global DNS -> Regional LB -> Stateless compute (multi-AZ)
                                -> Managed DB (multi-AZ)
                                -> Queue/Stream
                                -> Object Storage + CDN
```

## Practical example

Image upload product:
- store original in blob store
- serve via CDN
- metadata in DB
- async thumbnail worker via queue

## Real-world use cases
Startup scaling, cost optimization, disaster recovery planning.

## Interview questions

**Q. When do you go multi-AZ vs multi-region?**
When I need resilience to **datacenter/AZ-level failures** within a region, I use **multi-AZ** (low latency, simpler, common default). I go **multi-region** only when I must survive a **full region outage** or serve global users, because it adds major complexity—**cross-region replication, failover, higher latency, and tougher consistency trade-offs**.
 
**Q. What parts of your system are hardest to make multi-region and why?**
The hardest parts are the **stateful pieces**—DB/transactions, caches, streams/queues, and id generation—because multi-region introduces **replication lag, conflict resolution, ordering/consistency issues, and complex failover** (writes can’t be strongly consistent everywhere without latency trade-offs).
 
**Q. What costs surprise teams most at scale?**
Usually **data transfer/egress** (cross-AZ/cross-region + CDN miss traffic), plus **managed DB cost growth** (storage/IOPS/replicas/backups) and **observability costs** (logs/metrics/traces volume).