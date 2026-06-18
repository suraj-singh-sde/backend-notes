# Chapter 9 — Caching

> The secret behind almost every fast system. Caching trades a little freshness and memory for enormous gains in speed and cost.

---

## 1. What Is Caching?

- **Definition:** caching stores a subset of data in a location that's faster and easier to access than the original source.
- **Purpose:** it drastically cuts the time and compute needed to fetch data or perform an operation — essential for systems that need microsecond/millisecond latency.

The fundamental bet: **reading from cache is far cheaper than recomputing or refetching**, and most workloads ask for the same hot data over and over.

---

## 2. Real-World Examples of Caching

- **Google Search:** rather than re-running expensive ranking/indexing for every query, Google serves common queries ("what is the weather today") from a globally distributed in-memory cache. A match is a **cache hit** and returns instantly.
- **Netflix:** to stream terabytes globally without buffering, Netflix uses a **CDN** with **Edge servers** placed near users, caching optimized video files locally so requests don't travel back to the origin in the US.
- **Twitter/X (trending):** computing trends over billions of tweets is expensive, so it's recomputed every few minutes and the result cached in Redis — every user who taps "Trending" reads the same cached result.

---

## 3. The Three Levels of Caching

As a backend engineer you'll deal with caching at the **Network**, **Hardware**, and **Software** levels.

### A. Network-Level Caching

- **Content Delivery Networks (CDNs):** cache static resources (images, video, HTML/JS/CSS) on Edge nodes grouped into regions called **PoPs (Points of Presence)**.
  - The CDN's DNS routes a request to the closest PoP by geography and network speed.
  - Content present at the edge → **cache hit**, served immediately. Absent → **cache miss**, fetched from origin and then cached.
  - **TTL (Time to Live):** how long the edge keeps content before refetching a fresh copy.
- **DNS Caching:** translating a domain (`example.com`) to an IP queries several servers, so results are cached at multiple layers:
  1. **OS cache** — checked first.
  2. **Browser cache** — e.g., Chrome's own cache.
  3. **Recursive resolver cache** — your ISP's resolver (or Google Public DNS) checks its cache before recursively querying Root and TLD servers.

### B. Hardware-Level Caching

- **CPU caches:** L1/L2/L3 caches keep frequently used data and predictable sequences (e.g., array traversal) close to the CPU for fast access.
- **RAM vs. disk:** RAM accesses data via electrical signals — nearly instantaneous versus disks with mechanical parts. But RAM is volatile (cleared on power loss) and limited in capacity.

### C. Software-Level Caching (In-Memory Databases)

Technologies like **Redis**, **Memcached**, and **AWS ElastiCache** use RAM instead of disk for blazing-fast reads/writes.

- **Characteristics:** they're NoSQL **key-value** stores — data (strings, JSON, lists) is keyed by simple keys rather than relational tables. To avoid data loss, they often persist to disk in the background.

```
Latency ladder (rough orders of magnitude):
  CPU cache      ~1 ns
  RAM / Redis    ~100 ns – <1 ms
  SSD            ~100 µs
  Disk DB query  ~1–10 ms
  Network/origin ~50–200 ms
```

Each step down is roughly 10–1000× slower — which is exactly why we cache hot data higher up the ladder.

---

## 4. In-Memory Caching Strategies

### Cache-Aside (Lazy Caching)

The application caches data only after it's first requested. On a miss, it loads from the database, stores it in the cache, then returns it.

```
read(key):
  value = cache.get(key)
  if value is None:            # cache miss
      value = db.query(key)    # fetch from source of truth
      cache.set(key, value, ttl=300)
  return value
```

*Pro:* only caches what's actually used. *Con:* the first request per key is always a miss (a "cold" cache).

### Write-Through

Every write goes to the database **and** the cache at the same time, so the cache is always fresh.

```
write(key, value):
  db.save(key, value)
  cache.set(key, value)   # updated in lockstep — reads never see stale data
```

*Pro:* reads are always fresh. *Con:* adds latency to every write, and caches data that may never be read.

### Write-Back (Write-Behind)

Writes go to the cache first and are flushed to the database asynchronously later.

*Pro:* extremely fast writes. *Con:* risk of data loss if the cache dies before flushing — use only when some loss is tolerable.

---

## 5. Cache Eviction Policies

RAM is limited, so a full cache must evict old data to make room. The policy decides what goes:

| Policy | Evicts… | Good when |
|--------|---------|-----------|
| **No Eviction** | nothing — errors when full | you must never silently drop data |
| **LRU** (Least Recently Used) | the key untouched for the longest time | recent access predicts future access |
| **LFU** (Least Frequently Used) | the key accessed fewest times overall | popularity is stable over time |
| **TTL** (Time to Live) | any key past its expiry | data has a natural freshness window |

---

## 6. Cache Invalidation & Common Pitfalls

> "There are only two hard things in computer science: cache invalidation and naming things." Caching is easy to add and hard to keep *correct*.

- **Staleness:** once data is cached, an update to the source can leave the cache serving outdated data until it expires. Mitigate by **invalidating** (deleting) the key on write, or by choosing a TTL short enough that staleness is acceptable.
- **Cache stampede / thundering herd:** when a popular key expires, hundreds of simultaneous requests all miss and hammer the database at once — sometimes hard enough to crash it.
  - *Mitigations:* add **random jitter** to TTLs so keys don't all expire together; use a **lock** so only one request recomputes while others wait; or **refresh ahead** of expiry.
- **Cache penetration:** repeated requests for keys that don't exist skip the cache entirely and always hit the DB. Mitigate by caching a "not found" marker (with a short TTL).

```
Stampede:                          With a lock (single-flight):
key expires at t=0                 key expires at t=0
1000 requests → 1000 DB queries    1000 requests → 1 DB query, 999 wait for the result
        💥 DB overload                     ✔ DB protected
```

---

## 7. Top Use Cases in Backend Engineering

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

---

← Previous: [« Chapter 8 — Databases](08-databases.md) · Back to the [index](../README.md)
