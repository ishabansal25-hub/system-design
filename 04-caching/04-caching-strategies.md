# Caching Strategies

[← Back to Caching](./README.md) | [← Previous: Types of Caches](./03-types-of-caches.md)

---

There are five main patterns for how your app interacts with cache and database. Each has tradeoffs. Here's when to use what.

## Quick overview

| Strategy | How it works | Best for |
|----------|--------------|----------|
| Cache-Aside | App manages cache manually | Most use cases |
| Read-Through | Cache fetches from DB automatically | Read-heavy, simpler code |
| Write-Through | Writes go to cache, then DB (sync) | Need strong consistency |
| Write-Behind | Writes go to cache, DB updated async | Write-heavy, can tolerate lag |
| Write-Around | Writes skip cache entirely | Write-once data |

If you're not sure, **start with Cache-Aside**. It's the most common and gives you the most control.

---

## 1. Cache-Aside (Lazy Loading)

This is the one you'll use 80% of the time.

**How it works:**

On read:
1. App checks cache
2. If hit → return data
3. If miss → query DB, store in cache, return data

On write:
1. App updates DB
2. App deletes the cache key (invalidation)

```
READ:
App → "Is user:123 in cache?"
     → Yes? Return it.
     → No? Get from DB, put in cache, return it.

WRITE:
App → Update DB
    → Delete user:123 from cache
    → Next read will fetch fresh data
```

**Why delete instead of update on write?** Because of race conditions. If two writes happen at the same time, you might end up with stale data in cache. Deleting is safer - the next read will just fetch fresh data.

**The good:**
- Simple to understand and implement
- Cache only contains data that's actually requested
- If cache dies, app still works (just slower)

**The bad:**
- First request for any data is always slow (cache miss)
- Small window where cache can be stale
- Your app code has to handle all the caching logic

**When to use:** Default choice for most applications.

---

## 2. Read-Through

Similar to cache-aside, but the cache itself handles fetching from the database.

**How it works:**

On read:
1. App asks cache for data
2. Cache checks if it has it
3. If miss → cache fetches from DB, stores it, returns to app

The app never talks to the DB directly for reads. The cache is the middleman.

```
App → Cache → "Do you have user:123?"
              → Yes? Here it is.
              → No? Let me get it from DB... got it, here you go.
```

**The good:**
- Simpler application code (no cache logic)
- Consistent caching behavior

**The bad:**
- First request still slow
- Need a cache library that supports this pattern
- Less flexibility

**When to use:** Read-heavy workloads where you want cleaner code.

---

## 3. Write-Through

Writes go to cache first, then cache writes to DB synchronously.

**How it works:**

On write:
1. App writes to cache
2. Cache writes to DB (waits for confirmation)
3. Returns success to app

```
App → Cache → "Store user:123"
              → Cache stores it
              → Cache writes to DB
              → DB confirms
              → Cache confirms to App
```

**The good:**
- Cache always has the latest data
- No stale data window
- Read-after-write consistency

**The bad:**
- Writes are slower (two writes: cache + DB)
- Cache might store data that's never read
- More complex failure handling

**When to use:** When you absolutely need consistency and can tolerate slower writes. Financial data, inventory counts, anything where stale data causes real problems.

---

## 4. Write-Behind (Write-Back)

Writes go to cache, and DB is updated asynchronously later.

**How it works:**

On write:
1. App writes to cache
2. Cache immediately returns success
3. Cache queues the write
4. Background process writes to DB (maybe batched)

```
App → Cache → "Store user:123"
              → Cache stores it
              → "Done!" (returns immediately)
              
              ... later ...
              
              → Cache writes to DB in background
```

**The good:**
- Fastest write performance
- Can batch multiple writes together
- Absorbs write spikes

**The bad:**
- Data can be lost if cache crashes before DB write
- Eventual consistency only
- Complex to implement correctly
- Debugging is harder

**When to use:** Write-heavy workloads where you can tolerate some data loss. Analytics, logging, non-critical updates.

---

## 5. Write-Around

Writes go directly to DB, skipping the cache entirely.

**How it works:**

On write:
1. App writes directly to DB
2. Cache is not updated

On read:
1. Normal cache-aside pattern

```
WRITE:
App → DB (cache not involved)

READ:
App → Cache → miss → DB → store in cache → return
```

**The good:**
- Cache doesn't get polluted with write-once data
- Good for data that's written but rarely read

**The bad:**
- First read after write is always a cache miss
- Not suitable if you need to read data right after writing

**When to use:** Audit logs, historical data, anything that's written once and rarely accessed.

---

## How to choose

Here's my mental model:

```
Do you need read-after-write consistency?
├── Yes → Write-Through
└── No
    ├── Is it write-heavy?
    │   ├── Yes, and data loss is OK → Write-Behind
    │   └── Yes, but need durability → Write-Through
    └── Is it read-heavy?
        ├── Yes → Cache-Aside or Read-Through
        └── Mixed → Cache-Aside
        
Is the data written once and rarely read?
└── Yes → Write-Around
```

Or simpler: **just use Cache-Aside** unless you have a specific reason not to. It's the most flexible and easiest to reason about.

---

## Combining strategies

You can mix these. For example:

- **Cache-Aside for reads + Write-Through for critical writes**: Get the simplicity of cache-aside but ensure important data is always fresh.

- **Write-Behind for bulk operations + Cache-Aside for normal operations**: Batch your heavy writes but keep normal operations simple.

- **Local cache (Read-Through) + Distributed cache (Cache-Aside)**: Two-tier caching where local cache auto-fetches from Redis, and Redis uses cache-aside with DB.

Don't overcomplicate it though. Start simple, measure, then optimize.

---

## Common mistakes

**Updating cache instead of deleting on write.** This causes race conditions. Two concurrent writes can leave stale data in cache. Always delete.

**Using Write-Behind without understanding the risks.** If your cache crashes, you lose data. Make sure that's acceptable.

**Caching everything with Write-Through.** You'll cache data that's never read, wasting memory.

**Not handling cache failures.** What happens when Redis is down? Your app should degrade gracefully, not crash.

---

[Next: Eviction Policies →](./05-eviction-policies.md)