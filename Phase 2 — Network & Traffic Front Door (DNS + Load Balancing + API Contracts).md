### Outcome of Phase 2
You should be able to answer (and draw) in interviews:
- “How does traffic enter my system globally?”
- “How do I route to healthy instances and do safe rollouts?”
- “How do I design clean APIs with correct retry/error semantics?”

## 2.1 DNS (how users find you)

### Intuition
DNS is your system’s **entry routing table**. It controls **where traffic goes** and how fast you can **fail over**.

### Concepts (must-know)
- **Name → IP** resolution with caching at multiple layers (browser/OS/ISP/resolver).
- **TTL** (time-to-live): how long answers are cached.
    - Low TTL = faster failover, higher DNS load
    - High TTL = faster steady-state, slower failover
- Records you should know:
    - **A/AAAA** (IP), **CNAME** (alias), **NS** (nameserver)
	- **A / AAAA:** “give me the IP”
	- **CNAME:** “this name is an alias of that name”
- **NS:** “who is the boss DNS server for this domain?”

- DNS routing patterns:
    - **Weighted** (gradual rollout)
    - **Failover** (primary/secondary)
    - **Geo/Latency-based** (send users to nearest region)

### Architecture (where it fits)

```
User -> DNS -> (chooses region / entry IP) -> Global/Regional LB -> Services
```

### Practical example

**Weighted DNS rollout**: send 10% traffic to new stack.
- `api.example.com` → 90% old LB, 10% new LB (via DNS routing)

### Real-world use cases
- Multi-region routing
- Blue/green + canary via DNS (works, but slower due to caching)
- Fast mitigation during incidents (if TTL is reasonable)

### Interview questions

**Q. What TTL would you choose for an API domain and why?**
For an API domain I’d pick a **moderate TTL** (e.g., ~30–300s): low enough to enable reasonably fast failover/traffic shifts during incidents, but not so low that DNS query volume becomes a major dependency/cost; the exact value depends on how often we need DNS-based cutovers.
 
**Q. DNS-based failover vs global load balancer failover—trade-offs?**
* The global load balancer keeps the same DNS entry and **routes traffic from Region A to Region B** when Region A/service becomes unhealthy (per-request, fast).
* DNS failover means the **DNS answer changes** from **LB-A to LB-B**; users switch when their **DNS caches refresh/TTL expires**, so it’s slower and less predictable.

 DNS-based failover (LB-A → LB-B via DNS answer change)
- **Benefit:** Simple and cheap; works across regions/clouds with minimal extra infra.
- **Trade-off:** Failover is slower/less predictable due to **TTL + caching** (some users keep hitting LB-A until cache expires).

Global load balancer failover (DNS stays same, global LB reroutes per request)
- **Benefit:** Faster and more deterministic failover (seconds) with better health-based, per-request traffic steering.
- **Trade-off:** Higher cost and operational complexity; still needs a solid **multi-region data replication/consistency** plan.


## 2.2 Load Balancers (route traffic to healthy compute)

### Intuition
LBs protect your services from the chaos of the network and keep scaling simple: **add more instances; LB spreads traffic**.

### Concepts (must-know)
- **L4 vs L7**
    - **L4 (TCP/UDP):** fast, forwards connections. Works at connection level (IP + port).
    - **L7 (HTTP):** routes by host/path/header, can do auth/rate-limit/WAF
- **Algorithms**
    - Round-robin, least connections, weighted, consistent hashing (sticky-ish)
- **Health checks**
    - Pull unhealthy instances out automatically
- **Connection draining**
    - During deploys, you don’t want to kill a server while it’s handling requests.
    - Draining means:
		- stop sending new requests to that instance
		- let in-flight requests finish
		- then terminate it safely
- **Sticky sessions**
    - Avoid unless needed (breaks horizontal scaling + failover simplicity)

### Architecture (typical)

```
            +------------------+
Users ----> | Global / Edge LB |
            +------------------+
                      |
            +------------------+
            | Regional L7 LB   |
            +------------------+
                |        |
               v          v
          App A            App B   (stateless)
```
- A **global/edge LB** can route users to the right region (or nearest POP)
- A **regional L7 LB** routes to the correct service/instance in that region
- Apps are **stateless**, so LB can send requests to any instance
### Practical example
“New deploy causes errors” → LB removes instances failing readiness checks; traffic stabilizes.

### Real-world use cases
- Stateless API services, web apps
- Multi-AZ traffic distribution
- Zero-downtime rollouts

### Interview questions

**Q. What’s the difference between readiness and liveness for LB health checks?**
**Liveness** checks if the process is alive (fail → restart). **Readiness** checks if it can serve traffic (fail → remove from LB / stop routing).

**Q. When would you choose consistent hashing at the LB?**
Use **consistent hashing** when you must route by a stable key (e.g., **userId/tenantId**) to the same backend for **locality/sharded state** (cache/session/partitioned processing) while minimizing remaps on scale changes; avoid it for stateless services because it adds stickiness and complicates autoscaling/failover.


## 2.3 Edge / API Gateway layer (the “front door brain”)

### Intuition
Put cross-cutting concerns at the edge so every service doesn’t re-implement them.

### Concepts (must-know)
Common gateway responsibilities:
- **Authentication** (JWT verification / token introspection)
- **Authorization** (sometimes, but many keep authz in service)
- **Rate limiting / quotas** (global or per-tenant)
- **Request routing** (host/path → service)
- **Request normalization** (headers, compression). the gateway/edge makes incoming requests **consistent and safe** before they reach services, so every service doesn’t handle quirks or attacks differently.
- **Observability** creates (requestId/trace propagation). So you can trace one request across microservices.
- **WAF(Web Application Firewall) / DDoS basics** (often via managed edge services)

### Architecture
```
Client -> DNS -> LB -> API Gateway -> Services
                         |
                         +-> rate limit, auth, routing, logs
```

### Practical example
All requests get a `traceId` at gateway → every downstream log includes it.

### Real-world use cases
- Microservices platform
- Multi-tenant SaaS (per-tenant rate limits)
- Public APIs with abuse prevention

### Interview questions

**Q. What should be done at gateway vs inside services?**
Gateway handles shared cross-cutting concerns (authn, routing, coarse rate limits, WAF, normalization, tracing); services enforce domain correctness and fine-grained authorization with idempotency and transactional guarantees.

**Q. If gateway goes down, what’s the blast radius? How to reduce it?**
If the gateway is the single entry point, its outage makes **all external API traffic unavailable** (routing/auth/rate limiting/observability at the edge stop), while internal calls may still work. To reduce blast radius: run the gateway **multi-AZ with horizontal scaling + health checks**, keep config stateless, add **failover (global LB / secondary gateway)**, and ensure services have **defense-in-depth** (service-level auth + validation) so the gateway isn’t a single security point.



## 2.4 API Design & Contracts (implementation-friendly HLD)

### Intuition
**APIs are not just “endpoints”**—they’re **contracts** that protect you from failures, retries, and independent teams changing things.

A contract means:
- clients know exactly what to send/expect
- services can change internally without breaking clients
- retries and partial failures don’t create duplicates or corrupt data

### Concepts (must-know)

- **REST vs gRPC**
    - REST(HTTP + JSON): easy public APIs, caching friendliness, human-debuggable
    - gRPC(HTTP/2 + Protobuf): low latency, strict contracts, internal service-to-service

- **Idempotency**: Networks fail → clients retry → same request may reach you twice. So for endpoints like `POST /orders` or `POST /payments`, you must prevent duplicates.
    - Mandatory for “create payment/order” style endpoints
    - Use `Idempotency-Key` + dedupe storage

- **Pagination**
    - Prefer **cursor** pagination for feeds/search

- **Versioning**
    - version APIs and keep backward compatibility:
		- `/v1/...` (common)
		- or version in headers
		**Rule:** add fields as optional; don’t break existing fields.

- **Error model**
    - Instead of random error shapes, you standardize:
	    - stable `code` (machine readable)
		- human `message`
		- optional `details`
		- always include `requestId` so you can find logs/traces
		This makes debugging and client handling reliable.

- **Timeout budget discipline**: When a request goes through multiple hops (Gateway → Service → DB/other service), you must set **nested timeouts** so the outer layer has time to handle failures.

	**Rule:** **Caller timeout > Callee timeout**  
	(DB timeout < Service timeout < Gateway timeout < Client timeout)
	
	If inner timeouts exceed outer timeouts, your system keeps working on requests that are already “dead,” exhausting shared resources and causing cascading failures under load.
### Architecture pattern (contract-first)

 - **API Specification (OpenAPI / Proto)**: You first write the **formal contract** of your API:
	- **OpenAPI** (for REST/JSON): endpoints, request/response schemas, status codes, auth
	- **Proto** (for gRPC): service methods, request/response message types
	This spec is the **source of truth**.

- **Clients Generated:** From the spec, tools can auto-generate **client SDKs**:
	- typed functions like `createOrder(...)`, `getOrders(...)`
	- correct URL paths, headers, request body shapes
	- parsing responses into models
	**Benefit:** clients don’t “guess” the API shape → fewer integration bugs.

- **Server stubs generated**: From the same spec, you can generate **server skeleton code**:
	- the route/method signatures exist already
	- you just fill in the business logic

	**Benefit:** server implementation matches the contract by construction

- **Shared rules (error + pagination + idempotency)**: You also standardize common behaviors across services:
	- **Error model**: every service returns errors in the same JSON shape  
    (e.g., `{ code, message, details, requestId }`)
	- **Pagination model**: consistent format for lists  
	    (cursor/limit, nextCursor)
	- **Idempotency rules**: for POST “create” endpoints  
	    (`Idempotency-Key` header + dedupe behavior)

	**Benefit:** every service behaves predictably; clients handle responses uniformly.


- **RPC** stands for **Remote Procedure Call**.

	It means: **your code calls a function that runs on another machine/service**, as if it were a local function call.

	Example (mental model)
	Local call:
	- `getOrder(orderId)`
	
	RPC call:
	- your service calls `OrderService.GetOrder(orderId)`	    
	- but the function actually runs inside the **Order Service** over the network
	- it returns the result back to you

	 Why it’s used
	- Makes service-to-service communication feel like calling methods/functions
	- gRPC is a popular modern RPC framework
		
**One-liner:** RPC = “call a function on a remote service over the network.”


### Practical example (2 endpoints)
- `POST /v1/orders` (Idempotency-Key)
- `GET /v1/orders?cursor=...&limit=20`

### Real-world use cases
- Avoid breaking mobile clients
- Clean service boundaries in microservices
- Safe retries in flaky networks

### Interview questions

**Q. 409 vs 400: when and why?**
- **400 Bad Request:** the request itself is invalid (malformed JSON, missing/invalid fields, invalid enum/range).
- **409 Conflict:** the request is valid, but it can’t be applied due to a **state conflict** (optimistic lock/version mismatch, duplicate unique key, invalid state transition like canceling a shipped order).
 
**Q. How do you evolve an API without breaking old clients?**
We evolve APIs with **backward-compatible, additive changes**: don’t remove/rename fields, add new fields as optional, and ensure clients tolerate unknown fields. We **monitor usage and deprecate** old behavior over time, and only introduce a new version (`/v2` or header-based) for truly breaking changes.


## 2.5 “Traffic shaping” fundamentals (what SDE-3s mention)

_(Not full rate limiter design yet—just the edge concepts you should name.)_
- **Rate limit** (protect systems)
- **Load shedding** (drop non-critical work first)
- **Backpressure** (queues/stream, bounded concurrency)
- **Retries with backoff + jitter** (avoid retry storms)


## Phase 2 Integrated Reference Architecture (single-region + optional multi-region)

### Single region (most interview baselines)

```
Users
  |
 DNS
  |
L7 LB
  |
API Gateway (auth, rate limit, routing, tracing)
  |
Stateless Services  ---> Cache/DB/Queue (later phases)
```

### Multi-region (when asked “global”)

```
Users
  |
 Geo/Latency DNS
  |
Global LB (optional)
  |
Regional LB/API GW  -> Services (Multi-AZ)
```


## Phase 2 Drill (15-minute mini practice)

**Prompt:** “Design the traffic entry for a public API used by mobile clients globally.”  
Must cover:
- DNS strategy (geo/weighted/failover + TTL)
- LB layering (global vs regional, L7)
- Gateway responsibilities
- API contract basics (timeouts, idempotency, error model, tracing)