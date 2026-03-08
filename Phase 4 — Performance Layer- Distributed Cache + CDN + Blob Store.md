

## Outcome of Phase 4

You should be able to:
- decide **what to cache**, **where**, and **with what invalidation strategy**
- explain CDN routing + caching + cache consistency in interviews
- design a blob store pipeline (upload, download, metadata, lifecycle) that scales


# 4.1 Distributed Cache (Redis/Memcached style)

## 4.1.1 Intuition

Caches buy you **latency** and **cost reduction**, but they introduce **staleness** and **invalidation complexity**.  
In interviews, you should treat caching like a _performance optimization with correctness boundaries_.


## 4.1.2 Concepts (must-know)

### A) Where caches live
- **Client/browser cache** (HTTP caching)
- **CDN/edge cache** (static + media)
- **Service-side distributed cache** (Redis/Memcached)
- **In-process cache** (fastest, but per-instance and limited)

### B) Cache access patterns

**1) Cache-aside (lazy loading) — default**
- app checks cache
- on miss, read DB
- fill cache

```
Client -> Service
           |
           | get(key)
           v
         Cache
        /    \
   hit v      v miss
      return   DB -> return -> set cache
```

**2) Read-through**  
Cache itself fetches from DB on miss (less common in interviews unless asked).

**3) Write-through**  
Write goes to cache and DB together (simple reads, more write latency).

**4) Write-back (write-behind)**  
Write to cache first, DB async later (fast, risky—needs durability).


### C) Invalidation strategies (this is the “hard part”)

- **TTL-based**: simplest; accept brief staleness
- **Explicit invalidation**: on DB write, delete/update cache
- **Versioned keys**: `user:123:v7` so old caches naturally expire
- **Event-driven invalidation**: when DB changes: publish an event (`UserUpdated`), all services that cache that user invalidate their keys.

**Rule:** If correctness is critical (payments), don’t rely on cache for truth.


### D) Eviction + memory pressure

Caches have limited RAM. When full, they evict entries using policies:
- **LRU**: remove least recently used
- **LFU**: remove least frequently used

Risks:
- caching huge objects wastes memory
- eviction can cause sudden DB load spikes

Cache stampede (important)
Many requests miss at the same time (e.g., TTL expired) and all hit DB → DB overload.

Mitigations:
- **request coalescing (single-flight)**: only 1 request loads DB, others wait
- **soft TTL**: serve slightly stale + refresh in background
- **TTL jitter**: randomize expiry so not everything expires together


### E) Consistency risks
- stale reads
- thundering herds
- cache poisoning (security)
- split-brain in cache cluster (rare but mentionable)


## 4.1.3 Architecture patterns

### Cache-aside for DB-backed service (classic)
```
Client -> Service -> Cache
              \-> DB
```

### With stampede protection
```
Cache miss -> acquire lock(key) in Redis
   winner loads DB + sets cache
   others wait / serve stale
```



## 4.1.4 Practical example

**User profile service**
- Cache key: `user:{id}`
- TTL: 5–30 minutes
- On profile update: delete cache key (or bump version)
- Read path becomes ~1–5ms instead of DB 20–50ms


## 4.1.5 Real-world use cases

- session store, feature flags, catalog/product pages, feed precomputation, rate limiter counters    

## 4.1.6 Interview questions

**Q. Cache-aside vs write-through: when and why?**
When we want lower latency we prefer cache-aside and when we want DB and cache write update together(we want to avoid getting stale data from cache) we prefer write-through.
 
**Q. How do you prevent cache stampede for a hot key?**
We can do request coalescing where only 1 request loads data from the hot key while other waits.
 
**Q. What’s your invalidation strategy for “user profile updated”?**
We can do explicit invalidation like, on DB write: delete the cache key or update it. We can also used versioned keys, so when user changes and bumps into the older key, it becomes irrelevant. We can also implement event driven invalidation, like when a DB changes: we'll publish an event(`UserUpdated`). All services that cache that user invalidate their keys.



# 4.2 CDN (Content Delivery Network)

## 4.2.1 Intuition

A **CDN (Content Delivery Network)** is basically a **distributed cache** placed in many cities worldwide (edge locations).

It helps because:
- **Lower latency:** user gets content from a nearby edge server instead of your far-away origin.
- **Lower cost:** fewer requests hit your origin → less compute + less bandwidth (egress).
- **Spike protection:** if traffic jumps 10×, CDN absorbs most of it; origin doesn’t melt.

## 4.2.2 Concepts (must-know)

### A) What CDNs cache
- static assets: JS/CSS/images
- media: photos/videos
- sometimes API GET responses (careful with personalization)
- Rule: avoid caching anything user-specific unless you’re very careful (privacy leaks).

### B) Core behaviors (how CDN decides cache hit/miss)

#### Cache key
This is “how CDN identifies the object”.  
Usually built from:
- **URL path**
- sometimes **query params**
- sometimes selected **headers** (e.g., `Accept-Encoding`, language)

So:
- `/img/a.png?size=small` and `/img/a.png?size=large` can be different cache entries if query params are part of cache key.

#### TTL
How long edge keeps the object before treating it stale.

#### Revalidation (ETag / If-None-Match)
Instead of downloading full content again:
- client/CDN asks origin “has this changed?”
- origin replies **304 Not Modified** if same  
    Saves bandwidth.

#### Purge / invalidation
If you must update content immediately:
- you “purge” it from CDN so next request fetches fresh from origin.

#### Origin shielding
CDN adds an extra “mid-tier” cache layer so even cache misses don’t all hit your origin directly—reduces origin load further.

### C) Consistency vs performance
- you often accept eventual freshness for static/media
- for “must update instantly,” use purge/versioned URLs

**Best practice:** versioned assets: `/app.v123.js` → can be cached “forever”.


## 4.2.3 Architecture

```
User
  |
  v
Edge POP (CDN cache)
  |
  | miss
  v
Origin (LB -> Service / Blob store)
```


## 4.2.4 Practical example

Image delivery:
- Upload original to blob store
- Serve via CDN with `Cache-Control: public, max-age=31536000`
- Use versioned URLs or signed URLs


## 4.2.5 Real-world use cases

- websites, video streaming, app assets, global ecommerce images


## 4.2.6 Interview questions

**Q. How do you invalidate CDN caches after image update?**
Prefer **versioned URLs** (e.g., `/img.v123.png`) so caches update instantly; if the URL must stay same, **purge/invalidate** that path so the next request refetches from origin.
 
**Q. What should be in the cache key and what should not?**
Cache key should include **path + relevant query params** and only the headers that change representation (e.g., `Accept-Encoding`, sometimes `Accept-Language`); avoid **Authorization/cookies/user identifiers** unless intentionally doing private caching, otherwise you risk **data leaks** and **cache fragmentation**.
 
**Q. How do you protect origin during sudden traffic spikes?**
We can implement origin shielding by bringing in a mid-tier cache layer or a central cache layer in front of the origin so that edge cache misses don't all stampede the origin plus **rate limiting/WAF** to shed abusive spikes.



# 4.3 Blob Store (S3-like object storage)

## 4.3.1 Intuition

Blob store is optimized for:
- storing huge files cheaply and durably
- high throughput (especially downloads)  
    It is not for “querying”—you store metadata elsewhere.


## 4.3.2 Concepts (must-know)

### A) Split data vs metadata
- **Blob store:** actual bytes (images/videos/docs)
- **DB:** metadata (owner, size, content-type, checksum, permissions, status, blob_key)
 **Why split?**
- Blob store is cheap and scalable for bytes
- DB is queryable and transactional for metadata/state

### B) Upload patterns

**1) Direct upload via pre-signed URL (best)**
- client uploads straight to blob store
- your servers avoid bandwidth costs

Client -> Service (request upload)  
Service -> returns pre-signed URL  
Client -> Blob store (upload)  
Client -> Service (confirm/commit)


**2) Proxy upload through service**
Flow:
- Client uploads file to your API
- API forwards to S3

**Why it’s costly**
- You pay double bandwidth (client→service + service→S3)
- You bottleneck on your app servers  
    Use only for small files or very early prototypes.

### C) Multipart uploads

For big files (say >50MB, or videos):
- Split into chunks (e.g., 5–50MB parts)
- Upload parts in parallel
- Resume if network drops
- Final “complete multipart upload” merges parts

**Benefits**
- faster uploads (parallel)
- better reliability (resume)

### D) Consistency and safety

Uploads must be correct and secure.
- **Checksum validation**: confirm bytes are not corrupted  
    (MD5/SHA compare client vs server/S3 ETag rules)
- **Idempotent uploads**: retries shouldn’t create duplicate objects  
    Use an `upload_session_id` and deterministic object key
- **Virus scanning / content moderation** (enterprise)
    - mark file `PENDING_SCAN`
    - scan in background
    - only make it downloadable when `SAFE`

### E) Lifecycle management

To control cost and compliance:
- **TTL / expiry** (delete after X days)
- **hot → warm → cold** archival tiers (S3 standard → IA → Glacier)
- **retention policies / legal hold** (don’t delete for compliance)


## 4.3.3 Architecture (end-to-end blob pipeline)

```
          +-------------------+
Client -> | Upload Service    |
          +-------------------+
             | 1) create upload session
             v
          +-------------------+         +------------------+
          | Metadata DB       |<------->| Queue / Events   |
          +-------------------+         +------------------+
             | 2) presigned URL            |
             v                             v
          +-------------------+        +-------------------+
          | Blob Store        |        | Workers           |
          | (original bytes)  |        | thumbnail/scan    |
          +-------------------+        +-------------------+
             |
             v
          +-------------------+
          | CDN (download)    |
          +-------------------+
```

## 4.3.4 Practical example

“User uploads 3 room photos”
- create `upload_session`
- return 3 pre-signed PUT URLs
- client uploads
- worker generates thumbnails + stores derived variants
- CDN serves thumbnails/originals


## 4.3.5 Real-world use cases
- WhatsApp media ingestion, profile photos, invoices PDFs, video uploads, backups


## 4.3.6 Interview questions

**Q. Why store metadata in DB instead of blob store?**
 Because blob stores are great for **bytes**, not **querying/transactions**—you need a DB to **search/filter/sort** (by userId, time, status), enforce **constraints**, manage **permissions/status state machine** (UPLOADING→READY), and do **joins/indexes** reliably.
 
**Q. How do you prevent users from uploading huge files and killing costs?**
Enforce **max file size/content-type and per-tenant quotas** before issuing pre-signed URLs (and constrain upload size via signed policy where possible), plus **rate limits** and lifecycle cleanup for abandoned uploads to cap cost.
 
**Q. How do you make upload idempotent and resumable?**
Use an `upload_session_id` + deterministic object key, and for large files use **multipart upload** with stored `(partNumber, ETag)` checkpoints so clients can retry/resume parts; finalize via a **commit API** that is idempotent (dedupe on sessionId).


# Phase 4 Drill (20 minutes)

**Prompt:** “Design an image hosting service (upload + view).”  
Must cover:
- pre-signed upload flow
- metadata schema + permissions
- CDN delivery + cache headers
- cache strategy for metadata
- deep dive: invalidation + hot images + abuse (rate limit)