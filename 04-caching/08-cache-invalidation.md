# Cache Invalidation

[← Back to Caching](./README.md) | [← Previous: Distributed Caching](./07-distributed-caching.md)

---

> "There are only two hard things in Computer Science: cache invalidation and naming things."

This quote exists because invalidation is genuinely tricky. The cache has a copy of data, the database has the source of truth, and keeping them in sync is harder than it sounds.

---

## Why it's hard

The fundamental problem: you have data in two places (cache and database), and they can get out of sync.

**Scenarios that cause trouble:**

1. **Stale reads:** User A updates their profile. User B reads the cached (old) profile. User B sees outdated info.

2. **Race conditions:** Two users update the same record simultaneously. Depending on timing, the cache might end up with the wrong value.

3. **Distributed systems:** You have 10 app servers, each with local caches. User updates on server 1. Servers 2-10 still have old data.

4. **Failures:** You update the database, then try to invalidate cache, but the cache server is temporarily unreachable. Now they're out of sync.

There's no perfect solution. Every approach has tradeoffs.

---

## Strategy 1: TTL-Based Expiration

The simplest approach. Set a TTL, let data expire automatically.

```
SET user:123 {data} TTL=300  # Expires in 5 minutes

# After 5 minutes, key is gone
# Next read fetches fresh data from DB
```

**The tradeoff:** Data can be stale for up to TTL duration. If you set TTL=5min, users might see 5-minute-old data.

**When it works well:**
- Data that doesn't change often
- Data where slight staleness is acceptable
- When you want simplicity over perfect consistency

**Choosing TTL values:**

| Data | TTL | Reasoning |
|------|-----|-----------|
| User sessions | 30 min - 24 hours | Security vs convenience |
| Product prices | 5-15 minutes | Changes occasionally |
| User profiles | 1-5 minutes | Infrequent updates |
| Homepage content | 1-5 minutes | Can be slightly stale |
| Real-time data | Seconds | Needs freshness |

**Pro tip:** Add jitter to TTLs. Instead of all keys expiring at exactly 5 minutes, use random(4min, 6min). Prevents stampedes.

---

## Strategy 2: Explicit Invalidation

When data changes, explicitly delete (or update) the cache.

```python
def update_user(user_id, new_data):
    # 1. Update database
    db.update_user(user_id, new_data)
    
    # 2. Invalidate cache
    cache.delete(f"user:{user_id}")
    
    # Next read will fetch fresh data
```

**Delete vs Update:**

Always prefer **delete** over update. Here's why:

```
# BAD: Update cache
Thread A: Update DB to $100
Thread B: Update DB to $200
Thread B: Update cache to $200
Thread A: Update cache to $100  ← WRONG! Cache has stale data

# GOOD: Delete cache
Thread A: Update DB to $100
Thread B: Update DB to $200
Thread B: Delete cache
Thread A: Delete cache  ← Both delete, no problem
# Next read gets $200 from DB
```

Delete is idempotent. Multiple deletes are fine. Multiple updates can race.

---

## Strategy 3: Event-Driven Invalidation

Decouple the invalidation from the write path using events.

```
App updates DB
    ↓
DB publishes change event (CDC or app-level)
    ↓
Message queue (Kafka, etc.)
    ↓
Cache invalidation service consumes event
    ↓
Deletes from cache
```

**Why this helps:**
- Write path is fast (doesn't wait for cache)
- Invalidation can be retried if it fails
- Works across services

**The catch:**
- More infrastructure
- Eventual consistency (delay between DB write and cache invalidation)
- Message ordering issues

---

## Strategy 4: Version-Based Keys

Instead of invalidating, change the cache key.

```
# Version 1
Cache key: "user:123:v1"
Value: {name: "John", email: "old@email.com"}

# User updates email, version increments
Cache key: "user:123:v2"
Value: {name: "John", email: "new@email.com"}

# Old key (v1) just expires naturally
```

**How to track versions:**
- Store current version in a fast lookup (Redis itself, or DB)
- Or use a timestamp as version

**Why this works:**
- No race conditions (old and new versions are different keys)
- Easy rollback (just point back to old version)
- No explicit deletion needed

**The catch:**
- Old versions waste memory until they expire
- Need to track/lookup current version

---

## What I'd recommend

For most applications:

1. **Use TTL as a safety net.** Even with explicit invalidation, set a TTL. If invalidation fails, data eventually expires anyway.

2. **Delete on write.** When you update the database, delete the cache key. Don't try to update it.

3. **Accept eventual consistency.** For most use cases, a few seconds of staleness is fine. Don't over-engineer for perfect consistency unless you really need it.

4. **Keep it simple.** TTL + delete-on-write handles 90% of cases. Add event-driven invalidation only if you have specific requirements (cross-service, high reliability).

```python
def update_user(user_id, data):
    db.update(user_id, data)
    cache.delete(f"user:{user_id}")  # Simple and effective
```

---

## Common mistakes

**Forgetting to invalidate.** You add caching, it works great. Months later, someone adds a new update path and forgets to invalidate. Stale data bugs ensue.

**Invalidating before DB write.** If you delete cache, then DB write fails, you've invalidated for no reason. Always: DB first, then cache.

**Over-complicating.** Building elaborate pub/sub systems when TTL + delete would work fine.

**Not handling failures.** What if cache.delete() fails? At minimum, log it. Better: retry. Best: have TTL as backup.

---

[Next: Cache Stampede →](./09-cache-stampede.md)