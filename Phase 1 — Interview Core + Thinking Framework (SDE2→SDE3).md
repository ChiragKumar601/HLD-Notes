
## 1.1 The interview “control loop” (your default script)

**Intuition:** You must _drive_ the interview with a repeatable loop that forces clarity and surfaces trade-offs.  
**Concepts:** scope, assumptions, numbers, APIs, HLD, deep dives, failures, wrap.  
**Architecture output (what interviewer expects):**

```
Reqs + NFRs(Non Functional Req)
   -> Numbers
      -> APIs + Data model
         -> HLD (boxes/arrows)
            -> 2–3 deep dives
               -> Failures/ops
                  -> Trade-offs + wrap
```

**Practical example:** For “Design Notifications”, you explicitly choose: push/email/SMS? ordering? retries? user prefs?  
**Use cases:** writing RFCs, proposing system changes.  

**Practice Qs:**
- What are your default deep dives for _any_ system?
- How do you politely cap scope when interviewer keeps adding features?

## 1.2 Requirements: ask less, assume more (but document it)

**Intuition:** Requirements are your steering wheel. If you don’t lock them, you design the wrong system.  
**Concepts:** Functional vs NFR, MVP vs later, assumptions ledger.

**Fast requirement set (ask 5–7 max)**

1. Who are users? scale (DAU/MAU)?
2. Primary actions (top 3 flows)?
3. Read/write ratio? peak traffic?
4. Latency target (p95)?
5. Availability target (SLA-ish)?
6. Consistency expectations (strong vs eventual) for key actions
7. Global vs single region (and why)

**Assumptions ledger (say it out loud)**
- “I’ll assume peak read QPS = 10× avg.”
- “I’ll assume eventual consistency is fine for analytics, not for payments.”

**Practical example (URL shortener)**
- MVP: shorten, redirect, basic stats
- Later: custom domains, abuse ML, link expiration policies  
    **Use cases:** product scoping, roadmap decisions.  
    **Practice Qs:**
- What’s your “MVP boundary” for chat? for payments?
- How do you define “success metrics” in requirements?


## 1.3 NFRs that matter (and how to talk about them)

**Intuition:** SDE3 answers are “NFR-first”: they translate goals into concrete design decisions.  
**Concepts:** SLO(Service Layer Objective)/SLI(Service Layer Indicator), latency percentiles, availability, durability, scalability, maintainability.

## SLI / SLO (what they are)
- **SLI (Indicator):** the metric you measure  
    Examples: p95 latency, error rate, availability %, queue lag.
- **SLO (Objective):** the target for that metric  
    Examples: “p95 < 200ms”, “99.9% monthly availability”.
- **SLA:** external promise to customers (usually looser than SLO).

In interviews, you usually say: **“We’ll define SLIs and set SLOs for them.”**

## Availability vs Reliability (simple, interview-ready)
- **Availability:** did the system respond successfully right now? (uptime)
- **Reliability:** did it behave correctly over time? (no data loss, correct outcomes)

## The only latency numbers that matter

- **p50** (median), **p95**, **p99** (tail)
- Tail latency dominates user experience at scale.

### How NFRs change architecture (examples)
- “99.9% availability” → multi-AZ, health checks, graceful degradation
- “p95 < 200ms” → caching, indexes, avoid fanout on read path
- “no data loss” → durable queues, WAL, replicated storage

**Minimal SRE/ops checklist (say this near the end)**
- Dashboards: QPS, error rate, p95/p99 latency, saturation (CPU/mem), DB latency+conns, cache hit rate, queue lag
- Alerts: SLO burn, dependency failures, retry storms, DLQ growth

**Practical example:** For “Feed”, you accept eventual consistency for likes count; for “Payments”, you don’t.  
**Use cases:** on-call design, capacity planning.  
**Practice Qs:**
- For a cache outage, do you fail open or closed? when?
- How does “99.99%” change your design compared to “99.9%”?

## 1.4 Back-of-the-envelope (BOTE) — the 5 calculations you must do

**Intuition:** Numbers prevent fantasy architectures.  
**Concepts:** QPS, storage growth, bandwidth, hot keys, cost drivers.

### Core formulas
- **QPS(avg)** = requests/day ÷ 86,400
- **QPS(peak)** ≈ 5–10 × avg (unless told otherwise)
- **Storage/day** = If you store events, logs, messages, images metadata:
	events/day × bytes/event
	Why: tells you DB size, S3 cost, retention decisions.
- **Bandwidth** = How much data you’re sending over network: 
	QPS× avg response size
- **Cache memory** ≈ hot items × size/item × overhead
	Overhead includes metadata, key sizes, eviction structures. (So cache isn’t “exact bytes”, it’s more.)

Why: tells you if Redis fits or you need sharding/TTL.

### Quick “sanity constants”
- 1 day = 86,400 seconds
- p95 target implies: DB call budget often < 50–100ms if total p95 is 200ms
- Peak factor: assume 10× unless better info

**Mini example (numbers)**  
Assume: 10M redirects/day, response ~ 1KB
- QPS avg = 10,000,000 / 86,400 ≈ 115.7 QPS
- Peak QPS (10×) ≈ 1,157 QPS
- Bandwidth peak ≈ 1,157 KB/s ≈ ~1.1 MB/s (before overhead)

**Use cases:** deciding cache vs DB, single region vs multi.  
**Practice Qs:**
- Given 1B events/day at 200 bytes each, storage/month?
- If p99 spikes, what does it imply about queueing/saturation?


## 1.5 Phase 1 deliverable template (what you’ll produce every time)

Use this exact structure in interviews:
1. **Requirements** (MVP + explicit out-of-scope)
2. **NFR targets** (latency/availability/consistency)
3. **BOTE numbers** (QPS/storage/bw)
4. **APIs** (2–4 endpoints)
5. **Data model** (+ indexes)
6. **HLD diagram**
7. **Deep dives (2–3)**
8. **Failures + monitoring**
9. **Trade-offs + next steps**


## 1.6 Drill (Phase 1 exercise you should practice)

**Prompt:** “Design a URL shortener.”

Deliver in 12 minutes:
- 2 min requirements
- 2 min numbers
- 2 min APIs + schema
- 3 min HLD diagram
- 2 min deep dives (hot keys + caching)
- 1 min failures + wrap

**Deep dives to choose (default):**
- Hot key / popular URLs
- Storage + indexing
- Abuse + rate limiting (mention as extension)