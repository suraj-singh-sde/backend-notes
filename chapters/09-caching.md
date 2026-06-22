# Chapter 9 — Caching

> The secret behind almost every fast system. Caching trades a little freshness and memory for enormous gains in speed and cost.

---

## 1. What Is Caching?

- **Definition:** caching stores a subset of data in a location that's faster and easier to access than the original source (the **source of truth**).
- **Purpose:** it drastically cuts the time and compute needed to fetch data or perform an operation — essential for systems that need microsecond/millisecond latency.

The fundamental bet: **reading from cache is far cheaper than recomputing or refetching**, and most workloads ask for the same hot data over and over. This property is called **locality of reference** — data used recently (or used a lot) is likely to be used again soon.

Two terms you'll use constantly:

- **Cache hit:** the data was already in the cache → fast path, source untouched.
- **Cache miss:** it wasn't → you pay the full cost to fetch from the source, and usually store it for next time.

The single most important number in caching is the **hit ratio**:

```
hit ratio = hits / (hits + misses)
```

A cache at a **95%** hit ratio sends only 1 in 20 requests to your database. Pushing it to **99%** *cuts database load by 5×* — misses drop from 5% to 1%. This non-linearity is why a few points of hit ratio matter enormously, and why measuring it is non-negotiable (see §9).

> **The caching mindset — internalize this one rule:** a cache is an *optimization*, never a *source of truth*. The system must stay **correct** if the entire cache disappears — only slower. Almost every decision in this chapter (consistency, invalidation, failover) follows directly from that rule.

---

## 2. Real-World Examples of Caching

- **Google Search:** rather than re-running expensive ranking/indexing for every query, Google serves common queries ("what is the weather today") from a globally distributed in-memory cache. A match is a **cache hit** and returns instantly.
- **Netflix:** to stream terabytes globally without buffering, Netflix uses a **CDN** with **Edge servers** placed near users, caching optimized video files locally so requests don't travel back to the origin in the US.
- **Twitter/X (trending):** computing trends over billions of tweets is expensive, so it's recomputed every few minutes and the result cached in Redis — every user who taps "Trending" reads the same cached result.

Notice the shared shape: an **expensive computation or a distant fetch**, a **result reused by many requests**, and a **tolerance for slight staleness**. That triad is the signal that caching will pay off.

---

## 3. The Latency Ladder & The Three Levels of Caching

Caching only makes sense because every layer of a computer is dramatically slower than the one above it. Keep this ladder in your head:

```
Latency ladder (rough orders of magnitude):
  CPU register / L1 cache   ~1 ns
  L2 / L3 CPU cache         ~3–20 ns
  RAM (local process)       ~100 ns
  Redis/Memcached (network) ~0.2–1 ms   (RAM + a network hop)
  SSD                       ~100 µs
  Disk DB query             ~1–10 ms
  Cross-region / origin     ~50–200 ms
```

Each step down is roughly 10–1000× slower. Caching is simply **moving hot data up this ladder**. As a backend engineer you'll meet caching at three levels.

### A. Network-Level Caching

- **Content Delivery Networks (CDNs):** cache static resources (images, video, HTML/JS/CSS) on Edge nodes grouped into regions called **PoPs (Points of Presence)**.
  - The CDN's DNS routes a request to the closest PoP by geography and network speed.
  - Content present at the edge → **cache hit**, served immediately. Absent → **cache miss**, fetched from origin and then cached.
  - **TTL (Time to Live):** how long the edge keeps content before refetching a fresh copy. Controlled by HTTP headers — `Cache-Control: max-age=3600`, `ETag`, `Last-Modified` (ties back to HTTP caching in [Chapter 1](01-http.md)).
- **DNS Caching:** translating a domain (`example.com`) to an IP queries several servers, so results are cached at multiple layers:
  1. **OS cache** — checked first.
  2. **Browser cache** — e.g., Chrome's own cache.
  3. **Recursive resolver cache** — your ISP's resolver (or Google Public DNS) checks its cache before recursively querying Root and TLD servers.

### B. Hardware-Level Caching

- **CPU caches:** L1/L2/L3 caches keep frequently used data and predictable sequences (e.g., array traversal) close to the CPU. This is why iterating an array (contiguous, cache-friendly) is far faster than chasing pointers through a linked list (scattered, cache-hostile) — **memory layout is a performance feature.**
- **RAM vs. disk:** RAM accesses data via electrical signals — nearly instantaneous versus disks with mechanical parts. But RAM is volatile (cleared on power loss) and limited in capacity, which is exactly why a cache must never be the only copy of your data.

### C. Software-Level Caching (In-Memory Data Stores)

Technologies like **Redis**, **Memcached**, and **AWS ElastiCache** use RAM instead of disk for blazing-fast reads/writes.

- **Characteristics:** they're NoSQL **key-value** stores — data (strings, JSON, lists, sets, sorted sets, hashes) is keyed by simple keys rather than relational tables.
- **Redis vs. Memcached** — the two you'll choose between most often:

| | Redis | Memcached |
|---|---|---|
| Data types | Rich (strings, hashes, lists, sets, sorted sets, streams) | Strings/blobs only |
| Persistence | Optional (RDB snapshots, AOF log) | None — pure cache |
| Threading | Mostly single-threaded core | Multi-threaded |
| Built-in HA | Replication, Sentinel, Cluster | None (client-side sharding) |
| Pick it when | You want structures, persistence, pub/sub, atomic ops | You want a dead-simple, horizontally-sharded blob cache |

This chapter's examples use Redis-style commands, but the concepts apply to any in-memory cache.

---

## 4. Where the Cache Lives — Cache Topologies

*Before* choosing a strategy, decide **where** the cache physically sits. This single choice drives latency, consistency difficulty, and how you'll fight hot keys later.

- **In-process (local / L1):** a hash map or LRU structure *inside* your application process (e.g., Caffeine/Guava in Java, an in-memory `Map` in Node). Nanosecond access, zero network — but each app instance has its **own copy**, so caches drift out of sync and don't survive a restart.
- **Distributed (remote / L2):** a shared cache server like Redis that **all** app instances talk to over the network. One consistent copy, survives app restarts, scales independently — at the cost of a network hop and an extra system to operate.
- **Multi-tier / near-cache (L1 + L2):** a tiny local cache in front of the shared cache. Reads check L1 → L2 → database. Absorbs hot keys locally while keeping a shared L2 for everything else. This is the workhorse pattern at scale (see §10).
- **Sidecar:** a cache process co-located with each app instance (same host/pod) — local-cache latency without bloating the app's heap.

```
                          ┌── App A (L1 local) ──┐
   request ──▶ App tier ──┤                       ├──▶ Redis (L2 shared) ──▶ Database (truth)
                          └── App B (L1 local) ──┘
   nanoseconds  ◀── L1 ──▶  sub-millisecond ◀── L2 ──▶  milliseconds ◀── DB
```

| Topology | Latency | Consistency | Survives restart? | Best for |
|----------|---------|-------------|-------------------|----------|
| In-process (L1) | ~ns | **Hard** (per-instance copies drift) | No | Tiny, very hot, staleness-tolerant data |
| Distributed (L2) | sub-ms | Easier (one shared copy) | Yes | The default for shared application caching |
| Multi-tier (L1+L2) | ns→ms | Hardest (two layers to invalidate) | Partially | High-scale, hot-key-heavy workloads |

> **Trade-off in one line:** local caches are faster but harder to keep consistent; distributed caches are slightly slower but give you one truth to invalidate. Reach for multi-tier only once a shared cache alone can't absorb your hottest keys.

---

## 5. Caching Strategies (Read & Write Patterns)

A caching strategy answers two questions: **on a read, who fills the cache?** and **on a write, how do the cache and database stay in step?**

### Read patterns

**Cache-Aside (Lazy Loading)** — the application owns the cache. On a miss it loads from the DB, stores the result, and returns it. *The most common pattern by far.*

```
read(key):
  value = cache.get(key)
  if value is None:            # cache miss
      value = db.query(key)    # fetch from source of truth
      cache.set(key, value, ttl=300)
  return value
```

*Pro:* only caches what's actually used; the cache layer is dumb and replaceable. *Con:* the first request per key is always a miss (a "cold" key), and read/write code must agree on key format.

**Read-Through** — the cache itself knows how to load from the DB on a miss (via a configured loader). The app just calls `cache.get(key)` and never sees the database.

```
read(key):
  return cache.get(key)        # cache transparently loads from DB on a miss
```

*Pro:* loading logic lives in one place, not scattered across callers. *Con:* needs a cache library/layer that supports loaders; less control per call-site.

### Write patterns

**Write-Through** — every write goes to the cache **and** the database synchronously, so the cache is always fresh.

```
write(key, value):
  db.save(key, value)
  cache.set(key, value)   # updated in lockstep — reads never see stale data
```

*Pro:* reads are always fresh. *Con:* adds latency to every write, and caches data that may never be read.

**Write-Back (Write-Behind)** — writes go to the cache first and are flushed to the DB asynchronously (batched) later.

```
write(key, value):
  cache.set(key, value)        # acknowledged immediately
  queue.enqueue(key)           # background worker flushes to DB later
```

*Pro:* extremely fast writes; absorbs write bursts and batches them. *Con:* **risk of data loss** if the cache dies before flushing — use only when some loss is tolerable (e.g., view counters, metrics).

**Write-Around** — writes go **only** to the database, skipping the cache; the cache fills later on a read (via cache-aside).

```
write(key, value):
  db.save(key, value)          # cache is NOT written
  cache.delete(key)            # drop any stale copy so the next read reloads
```

*Pro:* avoids polluting the cache with write-heavy data that's rarely read again. *Con:* a freshly-written key is a guaranteed miss on its next read.

### When to use which

| Strategy | Type | Cache freshness | Best for | Watch out for |
|----------|------|-----------------|----------|---------------|
| **Cache-aside** | Read | Lazy | General-purpose, read-heavy | Cold-start misses; stampede on hot keys |
| **Read-through** | Read | Lazy | Centralizing load logic | Needs loader-aware cache layer |
| **Write-through** | Write | Always fresh | Read-after-write correctness | Write latency; caches cold data |
| **Write-back** | Write | Fresh, async-durable | Write bursts, counters | Data loss on cache crash |
| **Write-around** | Write | Filled on next read | Write-heavy, rarely re-read | Guaranteed miss right after a write |

> Most real systems combine **cache-aside reads + write-around (delete-on-write)**. It's simple, keeps the cache from filling with junk, and makes invalidation explicit. See why *delete* beats *update* in §7.

---

## 6. Cache Eviction Policies

RAM is limited, so a full cache must evict data to make room. The policy decides what goes:

| Policy | Evicts… | Good when |
|--------|---------|-----------|
| **No Eviction** | nothing — errors/rejects writes when full | you must never silently drop data |
| **LRU** (Least Recently Used) | the key untouched for the longest time | recent access predicts future access (the usual default) |
| **LFU** (Least Frequently Used) | the key accessed fewest times overall | popularity is stable over time |
| **FIFO** (First In, First Out) | the oldest-inserted key, regardless of use | insertion order ≈ relevance |
| **Random** | a random key | cheap; surprisingly fine when access is uniform |
| **TTL** (Time to Live) | any key past its expiry | data has a natural freshness window |

Eviction (cache is **full**) and expiration (a key's **TTL** elapsed) are different mechanisms that often work together: TTL bounds staleness; the eviction policy handles memory pressure.

**In practice (Redis `maxmemory-policy`):** you configure both a memory ceiling and a policy.

```
maxmemory 4gb
maxmemory-policy allkeys-lru      # evict the least-recently-used key from ALL keys

# Other common values:
#   noeviction      → reject writes when full (the cache becomes read-only)
#   allkeys-lfu     → least-frequently-used across all keys
#   volatile-lru    → LRU, but only among keys that have a TTL set
#   volatile-ttl    → evict the key with the nearest expiry first
```

> **Gotcha:** `volatile-*` policies only evict keys that have a TTL. If you set no TTLs and pick `volatile-lru`, a full cache has nothing eligible to evict and starts **rejecting writes** — a real production outage cause. Either set TTLs or use an `allkeys-*` policy.

---

## 7. Cache Invalidation & Consistency

> "There are only two hard things in computer science: cache invalidation and naming things." Caching is easy to add and hard to keep *correct*.

Once data is cached, an update to the source can leave the cache serving **stale** data. You have three ways to deal with it:

**1. TTL-based (expire and reload).** Give each key a TTL; accept staleness up to that window. Dead simple, no write-side coordination, eventually consistent. Use when "a few minutes old" is fine (product listings, trending, config).

**2. Explicit invalidation (delete on write).** On every write, delete the key so the next read reloads fresh.

```
update_product(id, data):
  db.update(id, data)
  cache.delete(f"product:{id}")   # next read repopulates from the DB
```

> **Why `delete`, not `set`?** Updating the cache directly invites a race: two concurrent writers can interleave so the cache ends up holding the *older* value. Deleting sidesteps it — the next reader simply reloads the current truth. **Prefer delete-on-write over update-in-place.**

**3. Versioned / namespaced keys.** Embed a version in the key; bump the version to invalidate a whole class of data at once, without scanning or deleting individual keys (the old keys age out via TTL).

```
key = f"user:{id}:profile:v{schema_version}"   # bump v to invalidate everyone instantly
```

### Coherence across many instances

With a **local (L1)** cache, each app instance holds its own copy. A write that invalidates one instance's copy leaves the others stale. Two fixes:

- **Short TTL on L1** — bound the staleness window cheaply (e.g., 5–30s).
- **Invalidation broadcast** — publish "key X changed" over a channel (e.g., Redis pub/sub) so every instance drops its local copy. More precise, more moving parts.

### The dual-write problem

Writing to the database and the cache are **two separate systems with no shared transaction**. A crash between them leaves them disagreeing:

```
write DB        ✔ committed
crash 💥
delete cache    ✗ never ran      →  cache now serves a stale value indefinitely
```

Mitigations, roughly in order of robustness:

- **Delete the cache *after* the DB commit**, and give every key a TTL as a safety net so staleness is bounded even if the delete is lost.
- **Read-through / write-through** so a single component owns both sides.
- **Change-Data-Capture (CDC):** drive invalidation from the database's commit log / binlog, so the cache is invalidated *because* the DB actually changed — not by hopeful application code. This is how the largest systems do it (see Meta in §10).

> **Rule of thumb:** never rely on the cache and DB being perfectly in sync. Bound staleness with a TTL, prefer delete-on-write, and design reads to tolerate a brief stale window.

---

## 8. Production Pitfalls & How to Survive Them

These are the failures that turn a cache from a speedup into the cause of an outage. Each is common enough to have a name.

### Cache stampede / thundering herd

When a popular key expires, hundreds of simultaneous requests all miss and hammer the database at once — sometimes hard enough to crash it.

```
Stampede:                          With a lock (single-flight):
key expires at t=0                 key expires at t=0
1000 requests → 1000 DB queries    1000 requests → 1 DB query, 999 wait for the result
        💥 DB overload                     ✔ DB protected
```

*Mitigations:*
- **Single-flight lock:** only one request recomputes the key; the rest wait for and reuse its result.
- **TTL jitter:** add randomness (`ttl = 300 + random(0..60)`) so hot keys don't all expire on the same tick.
- **Refresh-ahead:** asynchronously recompute a key *before* it expires, so reads never hit an empty slot.

### Cache avalanche

A bigger sibling of the stampede: a **large set** of keys expires simultaneously, or a **cache node dies**, so a flood of traffic falls through to the DB at once and collapses it.

*Mitigations:* spread TTLs with jitter; put a **circuit breaker / rate limiter** in front of the DB so the cache failing degrades latency instead of taking everything down; use a multi-tier cache so an L2 outage still has L1 absorbing some load.

### Cache penetration

Repeated requests for keys that **don't exist** skip the cache entirely (nothing to hit) and always reach the DB — a cheap way for an attacker, or a buggy client, to bypass your cache.

*Mitigations:*
- **Negative caching:** cache the "not found" result with a short TTL.

  ```
  value = db.query(key)
  cache.set(key, value if value else NOT_FOUND, ttl=30)   # cache misses too, briefly
  ```
- **Bloom filter:** a compact probabilistic set of "keys that could exist" — check it first and reject definitely-absent keys before touching the cache or DB.

### Hot key

One key (a celebrity profile, a viral tweet, a flash-sale product) gets so much traffic it saturates the **single shard/node** that owns it, even though the cluster as a whole is idle.

*Mitigations:* put a small **L1 local cache** in front so most reads never reach the shard; **replicate** the hot key across nodes and read a random replica; or **split** the value across several keys (`product:42:1`, `product:42:2`, …) and pick one per request.

### Big key

One key holds a huge value (a multi-megabyte list or hash). On single-threaded Redis, reading or deleting it **blocks every other request**, and it produces network spikes.

*Mitigations:* don't store giant blobs; **split** big collections into smaller keys; use **field-level operations** (`HGET` one field) instead of fetching the whole structure; delete large keys lazily/in chunks.

### Cold start (empty cache after deploy/restart)

A fresh or just-flushed cache has a **0% hit ratio** — every request misses and stampedes the DB right when you deploy.

*Mitigations:* **warm** the cache by preloading hot keys before taking traffic; roll out **gradually** so the herd ramps up; don't `FLUSHALL` on deploy; enable **persistence** (Redis RDB/AOF) so a restart reloads a warm dataset; keep a separate **warm pool** to fail over to (the regional-pool idea in §10).

---

## 9. Operating a Cache in Production

A cache is infrastructure — it needs availability, capacity planning, and monitoring like any datastore.

### High availability

- **Replication:** a primary with one or more read replicas. Replicas serve reads and stand by to take over.
- **Failover — Redis Sentinel:** monitors the primary and **automatically promotes** a replica if it dies. Use when one primary can hold your whole dataset but you need automatic failover.
- **Sharding — Redis Cluster:** partitions the keyspace across many primaries (each with replicas) so the dataset and throughput **scale horizontally**. Use when data or load outgrows a single node.

> **Sentinel vs. Cluster in one line:** Sentinel gives you *availability* for a single dataset; Cluster gives you *scale* by splitting the dataset. Big deployments use Cluster (which includes failover).

### Persistence (for Redis)

- **RDB:** periodic binary snapshots — fast restarts, but you can lose everything since the last snapshot.
- **AOF (Append-Only File):** logs every write — far less data loss, larger files and slower restart.
- For a **pure cache**, persistence is mainly about **avoiding a cold start** after a restart, not durability — the DB remains the source of truth.

### Capacity & memory

Set `maxmemory` + an eviction policy (§6), and watch **memory fragmentation** (RAM held by the allocator but not by live data). Size for your working set plus headroom, not your entire dataset.

### Observability — what to watch

You cannot operate what you don't measure. Track at minimum:

| Metric | Why it matters |
|--------|----------------|
| **Hit ratio** | The headline number. A falling ratio means the cache is doing less work and the DB more. |
| **Latency (p99)** | Tail latency reveals hot keys, big keys, and saturation that averages hide. |
| **Evictions/sec** | High evictions = the cache is too small for the working set. |
| **Memory used / fragmentation** | Predicts the next capacity problem before it pages you. |
| **Connections** | Connection storms (no pooling — see [Chapter 8](08-databases.md)) can exhaust the server. |

> **When NOT to cache.** Caching is not free — it adds a system, a consistency problem, and latency on misses. Skip it when: the **hit ratio is low** (data is rarely re-read), the workload is **write-heavy** (you invalidate as fast as you fill), or values are **huge or change every read**. A low-hit-ratio cache is pure overhead — measure before you add one, and be willing to remove it.

---

## 10. How Large Companies Cache

The patterns above, taken to their limit. Each entry pairs a **production pattern** with a company known for it.

- **Multi-tier / near-cache** — *pattern:* a tiny in-process L1 in front of a shared L2 (Redis/Memcached), so the hottest keys are answered locally and never hit the network. *Used widely* (Netflix, Meta, most large Java shops via Caffeine/Guava). *Trick:* keep a very short L1 TTL to bound staleness while still absorbing hot keys — exactly the hot-key defense from §8.

- **Consistent hashing & sharding** — *pattern:* spread keys across many cache nodes via consistent hashing, so adding or removing a node only remaps a **small fraction** of keys instead of reshuffling everything. *Example:* Memcached client libraries and **Twitter's Twemproxy ("nutcracker")** proxy; the idea traces back to Amazon's Dynamo and Akamai. *Trick:* it makes the cache tier **elastic** — scale out under load without a global cache wipe (which would cause an avalanche, §8).

- **Cache coherence with leases + a dedicated graph cache** — *example:* **Meta/Facebook.** Their *"Scaling Memcache at Facebook"* work introduced **leases** (a token that lets exactly one client recompute a key — solving stampede *and* stale-set races at once), **regional pools** and **cold-cluster warmup** (a new cluster reads from a warm one instead of stampeding the DB), and **invalidation driven from the MySQL commit log** (CDC, §7) so the cache is invalidated *because* the DB really changed. **TAO** is their read-through/write-through cache purpose-built for the **social graph** on top of MySQL.

- **Dedicated cache fleets across availability zones** — *example:* **Netflix EVCache**, a memcached-based tier that keeps **multiple copies across AZs**. Reads hit the nearest copy for low latency, and a whole AZ can fail without a cache outage. It fronts precomputed personalization/recommendation data so member home screens render fast.

- **Purpose-built cache servers** — *example:* **Twitter/X** built **Pelikan** (a framework for building cache servers) and **Nighthawk** (sharded, replicated Redis) because at their scale, off-the-shelf caches needed tighter control over memory, latency, and resharding.

- **Edge caching (CDNs)** — *pattern:* cache responses at PoPs near users (ties back to §3). *Example:* **Cloudflare, Akamai, Fastly.** *Trick:* push not just static assets but cacheable API responses to the edge, so most traffic is served thousands of miles from the origin — the lowest-latency cache of all is the one closest to the user.

> The throughline: at scale, caching becomes a **tiered, sharded, replicated, change-data-capture-driven** system in its own right — but every piece is still just an application of the fundamentals in §1–§9.

---

## 11. Top Use Cases in Backend Engineering

1. **Database query caching:** cache the results of compute-heavy queries (complex joins) or read-heavy endpoints (static product pages, celebrity profiles) so the primary DB doesn't buckle under traffic.

   ```
   GET /products/42 → cache miss → DB query → SET product:42 <json> EX 3600 → return
   GET /products/42 → cache hit  → return cached JSON (no DB hit)
   ```

2. **Storing user sessions:** after login, the session/auth token is stored in Redis (ties back to stateful auth in [Chapter 4](04-authentication-authorization.md)), giving fast verification on every request without touching the main DB.

   ```
   SET session:abc123 '{"userId":"u1","role":"admin"}' EX 1800   # 30-min session
   GET session:abc123                                            # validated in microseconds
   ```

3. **External API caching:** cache a third-party response (e.g., a weather API) with a TTL to save money on billing and avoid hitting their rate limits.

4. **Rate limiting:** a middleware extracts the client's IP, increments a Redis counter, and blocks with `429 Too Many Requests` past the limit — far faster than a relational DB (ties back to the rate-limiting middleware in [Chapter 6](06-layered-architecture.md)).

   ```
   INCR   rate:ip:203.0.113.5        # → 1 on first request
   EXPIRE rate:ip:203.0.113.5 60     # window resets after 60s
   # if INCR returns > 50  →  reject with 429
   ```

5. **Computed/aggregated results:** leaderboards, trending lists, and dashboards are expensive to compute over and over. Recompute on a schedule and serve everyone the cached result (Redis sorted sets are tailor-made for leaderboards).

---

← Previous: [« Chapter 8 — Databases](08-databases.md) · Back to the [index](../README.md)
