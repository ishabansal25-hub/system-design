# Caching Cheatsheet

Quick reference for interviews. Not comprehensive - just the stuff that comes up most.

---

## The basics (memorize these)

| Term | What it means |
|------|---------------|
| Cache Hit | Found in cache |
| Cache Miss | Not in cache, had to fetch |
| Hit Ratio | % of requests served from cache (aim for >90%) |
| TTL | Time To Live - when data expires |
| Eviction | Removing data to make room |

---

## Cache types

| Type | Where | Use case |
|------|-------|----------|
| Browser | Client | Static assets |
| CDN | Edge servers | Images, videos, JS/CSS |
| In-memory | App server | Hot data, sessions |
| Distributed | Redis/Memcached | Shared state |
| Database | DB buffer | Query results |

---

## The 5 strategies

**Cache-Aside** (use this by default)
```
Read: App → Cache → miss → DB → store in cache
Write: App → DB → delete from cache
```

**Read-Through**
```
Read: App → Cache (cache auto-fetches from DB on miss)
```

**Write-Through**
```
Write: App → Cache → DB (synchronous, both must succeed)
```

**Write-Behind**
```
Write: App → Cache → return immediately → DB updated async later
```

**Write-Around**
```
Write: App → DB (skip cache)
Read: Normal cache-aside
```

**When to use what:**
- Default → Cache-Aside
- Need consistency → Write-Through
- Write-heavy, can lose data → Write-Behind
- Write-once data → Write-Around

---

## Eviction policies

| Policy | Evicts | Use when |
|--------|--------|----------|
| LRU | Least Recently Used | Default choice |
| LFU | Least Frequently Used | Stable access patterns |
| FIFO | First In First Out | Simple cases |
| TTL | Expired items | Time-sensitive data |

LRU is almost always the right answer in interviews.

---

## Redis vs Memcached

| | Redis | Memcached |
|---|---|---|
| Data types | Lists, sets, hashes, etc. | Strings only |
| Persistence | Yes | No |
| Replication | Yes | No |
| Threading | Single (mostly) | Multi |

**Pick Redis** unless you have a specific reason for Memcached.

---

## Common problems

**Stampede (thundering herd)**
- Hot key expires, thousands of requests hit DB
- Fix: Locking, request coalescing, background refresh

**Cache penetration**
- Requests for non-existent data always hit DB
- Fix: Cache null results, bloom filter

**Inconsistency**
- Cache has stale data
- Fix: Delete on write (not update), short TTLs

**Cold start**
- Empty cache after restart
- Fix: Cache warming, gradual traffic shift

---

## Numbers worth knowing

| What | Latency |
|------|---------|
| L1 cache | ~0.5 ns |
| RAM | ~100 ns |
| Redis | ~1 ms |
| Database | ~10-100 ms |
| Network | ~100-200 ms |

Redis is ~100x faster than DB. That's why we cache.

---

## Key naming

Format: `{entity}:{id}:{field}`

```
user:123:profile
product:456:price
session:abc123
```

Keep them short but readable.

---

## Common mistakes

1. Caching everything (low hit ratio, wasted memory)
2. Long TTLs without invalidation (stale data)
3. Updating cache instead of deleting (race conditions)
4. No fallback when cache fails (system breaks)
5. Using cache as primary storage (data loss)

---

## Quick decision tree

```
What's your read:write ratio?
├── Mostly reads → Cache-Aside + LRU + TTL
├── Mostly writes → Consider Write-Behind
└── Need strong consistency → Write-Through

What's your scale?
├── Single server → Local cache (Guava/Caffeine)
├── Multiple servers → Distributed (Redis)
└── Both → Two-tier (local + Redis)
```

---

## Sample interview answer

**Q: "How would you cache a product catalog?"**

"I'd use Cache-Aside with Redis. On reads, check Redis first - if miss, fetch from DB and cache with a 5-10 minute TTL since prices can change. On writes, update DB then delete the cache key - delete instead of update to avoid race conditions.

For hot products, I'd add background refresh to prevent stampedes when popular items expire. I'd also implement a circuit breaker so if Redis goes down, we fall back to DB-only mode instead of failing completely.

Target hit ratio would be 90%+, which should reduce DB load by about 10x."

---

That's it. The rest is details you can look up.

[← Back to full guide](./README.md)