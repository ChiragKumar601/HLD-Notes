
## Outcome of Phase 5
You should be able to:
- decide **sync vs async** for any feature
- design a **distributed queue** with retries/DLQ/consumer scaling
- design **pub-sub** fanout patterns
- talk about **delivery guarantees** and **exactly-once effects**
- place streams (Kafka-like) correctly vs queues


# 5.1 Why async exists (the decision framework)

## Intuition
Async is how systems:
- absorb spikes (buffering)
- decouple slow/fragile dependencies
- improve user latency (return early)
- enable pipelines (event-driven architectures)

But async introduces:
- eventual consistency
- retries/duplicates
- ordering complexity
- operational overhead (lag, DLQ)

## Decision rule (interview-friendly)

Use **sync** when the user needs immediate truth now (login, payment authorization).  
Use **async** when work is heavy/slow/optional (email, thumbnails, analytics, search indexing).


# 5.2 Distributed Messaging Queue (RabbitMQ/SQS-like)

## 5.2.1 Intuition

A queue is a **buffer + work distributor**:
- producers push tasks
- consumers pull tasks
- system smooths load and provides retry handling


## 5.2.2 Core concepts (must-know)

### A) Basic semantics

- **At-least-once delivery** (most common): messages may be duplicated
- **At-most-once**: no duplicates, but may drop messages
- **Exactly-once delivery**: hard; usually aim for **exactly-once effects**

### B) Message lifecycle

```
Producer -> Queue -> Consumer
                 -> ack success (delete)
                 -> nack/fail (retry later)
                 -> exceed retries -> DLQ
```
- **ACK** = “done successfully, remove it”    
- **NACK/fail** = “couldn’t process, retry later”
- **DLQ** (dead-letter queue) = “this message keeps failing; isolate it for debugging

### C) Visibility timeout (SQS-style)

This is SQS’s key mechanism.
- Consumer receives message → message becomes **invisible** to others for a time window.
- If consumer **acks** before timeout → message deleted ✅
- If consumer crashes / doesn’t ack → timeout expires → message becomes visible again → **another consumer can process it** → duplicates possible.

So visibility timeout is why you must handle duplicates.

### D) Ordering

- strict ordering is expensive at scale
- common approach: order per key/partition (e.g., per userId)

### E) Backpressure

Queue backlog (length/lag) is your “pressure gauge”:
- if backlog grows, consumers are slower than producers
- you react by:
    - adding consumers (scale out)
    - slowing producers (rate limit)
    - or reducing work (degradation)


## 5.2.3 High-level architecture

```
+----------+      +-----------------+      +-----------+
| Producer | ---> | Queue/Broker     | ---> | Consumers |
+----------+      | partitions/acks  |      +-----------+
                  +-----------------+
                      |
                      v
                    DLQ
```


## 5.2.4 Detailed design highlights (what interviewers like)

### A) Partitioning the queue (scale)

- split into **partitions/shards**
- consumers in a group pull from partitions
- message routing:
    - round-robin (uniform)
    - key-based (preserve per-key ordering)

### B) Retry policy

- exponential backoff + jitter
- max retries
- poison message handling → DLQ

### C) Idempotent consumer (exactly-once effects)

Common patterns:
- **dedupe table**: store `messageId` processed
- **idempotency key**: use domain idempotency (orderId/eventId)
- **transactional outbox** (ties DB write + event publish safely)


## 5.2.5 Practical example

Order placed:
- sync: create order in DB
- async: send confirmation email + push event to analytics + update search index


## 5.2.6 Real-world use cases

- background jobs, email/SMS, media processing, billing, data pipelines


## 5.2.7 Interview questions

**Q. How do you handle poison messages?**
We can use bounded retries with exponential backoff with jittering and after max retries we can push the poison message or event to DLQ.
 
**Q. What makes a consumer idempotent?**
Idempotency Key + dedupe store is the key. Side effect guarded by unique constraint/state machine; ack after the commit.

**Q. How do you guarantee ordering for “per user” notifications?**
Strict order can be expensive but we can order per partition/per key. We may use for example, a user_id in a partition of a queue for ordering but ordering will be maintained in that partition only.




# 5.3 Pub-Sub (fanout communication pattern)

## 5.3.1 Intuition

Pub-sub is for **broadcasting events** to many subscribers without the publisher knowing who they are.

Queue = “do this job once”  
Pub-sub = “tell everyone this happened”


## 5.3.2 Concepts (must-know)

- **Topic**: named channel/stream of events. Example: `UserEvents`, `OrderEvents`, `PaymentEvents`.
- **Subscribers**: Subscribers are independent consumers that each get their **own copy** of events. Example: analytics service, cache invalidator, search indexer.
- **Delivery**: Most pub-sub systems deliver events **at least once**, so duplicates are possible. Subscribers must be **idempotent** (dedupe by eventId).
- **Fanout** patterns:
    - **Push-based:** broker pushes events to subscribers  
    (e.g., webhook style, or SNS to HTTP endpoints)
	- **Pull-based:** subscribers pull from their subscription/stream  
    (e.g., Kafka consumer groups, SQS subscriptions)
- **Filtering**:
    - topic per event type
    - attribute-based filtering


## 5.3.3 Architecture

```
          +-----------+
Publisher | Service A |
          +-----------+
                |
                v
           +---------+     +-----------+
           | Topic   | --> | Sub B     |
           +---------+ --> | Sub C     |
                        -->| Sub D     |
```


## 5.3.4 Practical example

User updates profile once → you publish a single `UserUpdated` event.
Different subscribers react:
- **Cache invalidator**: delete `user:{id}` from Redis
- **Search indexer**: update user document in search index
- **Analytics**: log profile update event
- **Recommendations**: update feature store

This avoids Service A calling 4 services synchronously.


## 5.3.5 Real-world use cases

- Microservices integration (loose coupling)
- Event-driven workflows (pipelines)
- Audit/event logs (record everything that happened)


## 5.3.6 Interview questions

**Q. Queue vs pub-sub: which would you use for sending emails? why?**
Use a **queue** for emails because it’s a background job with retries/DLQ and one job should be processed once; use **pub-sub** only if the same ‘EmailRequested’ event must be consumed by multiple independent systems (analytics/audit/etc.).

**Q. How do you prevent one slow subscriber from impacting others?**
Give each subscriber an **independent subscription/queue** (fanout), so one subscriber’s lag doesn’t block others; monitor lag and use DLQ/backpressure per subscriber.

**Q. How do you handle schema evolution of events?**
Use **backward-compatible, additive changes**: add optional fields, never remove/rename existing fields, version the schema (or include `schemaVersion`), and keep consumers tolerant of unknown fields; for breaking changes publish a new event version/topic (e.g., `UserUpdatedV2`) and migrate consumers gradually

# 5.4 Streams (Kafka-like) vs Queues (SQS-like)

## Intuition

Streams are **durable ordered logs** designed for:
- replay
- multiple consumer groups
- high throughput  

Queues are designed for:
- task distribution
- per-message ack/delete

### Key differences (interview-ready)

- **Retention**:
    - **Stream:** keeps data for X days/GB; consumers track offsets  
    → events are available later
	- **Queue:** typically removed after ack  
    → history is not kept (unless DLQ/custom archiving)

- **Replay**:
    - **Stream:** easy replay: “reset offset to yesterday”  
    → rebuild search index, re-run analytics after bug fix
	- **Queue:** replay is harder: once deleted, it’s gone  
    → you’d need DLQ, a separate audit store, or re-publish

- **Ordering**:
    - **Stream:** ordering is guaranteed **within a partition**  
    (key-based partitioning gives per-key order)
	- **Queue:** ordering varies:
	    - some provide limited ordering (SQS FIFO per group)
	    - standard queues don’t guarantee strict order

- **Consumer model**:
    - **Stream:** consumer groups + offsets  
    → multiple groups can consume independently
	- **Queue:** workers + ack/visibility timeout  
    → typically one processing pipeline per queue


# 5.5 Delivery guarantees + exactly-once effects (the essential truth)

## Intuition

You usually can’t avoid duplicates; you design so duplicates are harmless.

## Concepts

- **At least once** ⇒ duplicates possible
- **Exactly-once effects** ⇒ consumer is idempotent + dedupe + transactional patterns

**Transactional outbox (classic pattern)**
- write business change + outbox row in same DB transaction
- separate relay publishes outbox to broker
- prevents “DB committed but event not published” gap


# Phase 5 Reference Architecture (event-driven)

```
Client -> Service (sync DB write)
           |
           v
        Outbox Table  -> Outbox Relay -> Topic/Queue -> Consumers
                                            |
                                            v
                                           DLQ
```



# Phase 5 Drill (20 minutes)

**Prompt:** “Design an email notification system.”  
Must cover:

- queue vs pub-sub decision
- retry + DLQ policy
- idempotent sending (dedupe)
- rate limits with provider
- monitoring: queue lag, send success rate, bounce rate