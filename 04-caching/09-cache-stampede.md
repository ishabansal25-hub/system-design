# Cache Stampede

[← Back to Caching](./README.md) | [← Previous: Cache Invalidation](./08-cache-invalidation.md)

---

Also called "thundering herd" or "dog-pile effect." It's when a popular cache key expires and suddenly thousands of requests all try to rebuild it at once.

## The problem

```
Normal operation:
- Cache key "homepage_data" serves 10,000 requests/second
- Database load: ~0 (everything from cache)

Key expires:
- 10,000 requests simultaneously see "cache miss"
- 10,000 requests all query the database
- Database: "I can't handle this"
- System goes down
```

This is especially bad for:
- Hot keys (popular data accessed by many users)
- Expensive queries (the ones you cached because they're slow)
- Synchronized TTLs (many keys expiring at the same time)

---

## Solution 1: Locking

Only one request fetches from the database. Others wait for it.

```python
def get_data(key):
    data = cache.get(key)
    if data:
        return data
    
    # Try to acquire lock
    lock_key = f"lock:{key}"
    if cache.set(lock_key, "1", nx=True, ex=10):  # nx=only if not exists
        try:
            # I got the lock, I'll fetch
            data = db.query(key)
            cache.set(key, data, ex=300)
            return data
        finally:
            cache.delete(lock_key)
    else:
        # Someone else is fetching, wait and retry
        time.sleep(0.1)
        return get_data(key)  # Retry
```

**The good:** Guarantees only one database query.

**The bad:** 
- Waiting requests add latency
- Need to handle lock timeouts (what if the lock holder crashes?)
- Potential for deadlocks if not careful

---

## Solution 2: Request Coalescing (Singleflight)

Similar to locking, but built into the application layer. Multiple concurrent requests for the same key get grouped - only one actually fetches, others wait for its result.

```go
// Go's singleflight package does this elegantly
var group singleflight.Group

func getData(key string) (interface{}, error) {
    // All concurrent calls for same key share one fetch
    result, err, _ := group.Do(key, func() (interface{}, error) {
        return db.Query(key)
    })
    return result, err
}
```

```
Time 0ms: Request 1 for "user:123" → starts fetching
Time 1ms: Request 2 for "user:123" → joins Request 1
Time 2ms: Request 3 for "user:123" → joins Request 1
Time 100ms: Fetch complete → all 3 get the same result
```

**The good:** 
- Elegant, no explicit locks
- Built into many languages/frameworks

**The bad:**
- Only works within a single process
- For distributed systems, you need distributed locking

This is my go-to solution for most cases.

---

## Solution 3: Probabilistic Early Refresh

Instead of all requests seeing expiration at the same moment, randomly refresh before expiration.

```python
def get_data(key):
    data, ttl_remaining = cache.get_with_ttl(key)
    
    if data is None:
        # Actually expired, must fetch
        return fetch_and_cache(key)
    
    # Probabilistic early refresh
    # As TTL approaches 0, probability of refresh increases
    if should_refresh_early(ttl_remaining, total_ttl=300):
        # Refresh in background, return stale data now
        background_refresh(key)
    
    return data

def should_refresh_early(ttl_remaining, total_ttl):
    # More likely to refresh as expiration approaches
    # At 10% TTL remaining, ~50% chance to refresh
    threshold = total_ttl * 0.1
    if ttl_remaining < threshold:
        probability = 1 - (ttl_remaining / threshold)
        return random.random() < probability
    return False
```

**The good:**
- No waiting, no locks
- Spreads refresh load over time

**The bad:**
- Probabilistic (not guaranteed)
- Might serve slightly stale data during refresh

---

## Solution 4: Background Refresh

Don't wait for expiration. Proactively refresh hot keys before they expire.

```python
# Background job that runs continuously
def refresh_hot_keys():
    while True:
        for key in get_hot_keys():
            ttl = cache.ttl(key)
            if ttl < 60:  # Less than 1 minute remaining
                data = db.query(key)
                cache.set(key, data, ex=300)
        time.sleep(10)
```

**The good:**
- Users never see cache miss
- No stampede possible
- Predictable database load

**The bad:**
- Need to track which keys are "hot"
- Might refresh data that nobody's requesting anymore
- Extra infrastructure (background workers)

---

## Solution 5: Never Expire (with async refresh)

For truly critical data, don't use TTL at all. Always serve from cache, refresh asynchronously.

```python
def get_data(key):
    data = cache.get(key)
    
    if data is None:
        # Cold start - must fetch synchronously
        data = db.query(key)
        cache.set(key, data)  # No TTL
        return data
    
    # Check if refresh needed (store last_refresh timestamp in data)
    if needs_refresh(data):
        trigger_async_refresh(key)  # Don't wait
    
    return data  # Always return cached data
```

**The good:**
- Cache always has data
- No stampede possible

**The bad:**
- Data can be arbitrarily stale
- Need separate mechanism to trigger refreshes
- Memory usage (nothing ever expires automatically)

---

## What I'd use

**For most cases:** Request coalescing (singleflight pattern). It's simple, effective, and handles the common case well.

**For hot keys you know about:** Background refresh. If you know certain keys are critical, proactively refresh them.

**For distributed systems:** Distributed locking with Redis, combined with short lock TTLs and fallback to stale data.

**Combination approach:**
```python
def get_data(key):
    # 1. Try cache
    data = cache.get(key)
    if data:
        return data
    
    # 2. Try to get lock (with timeout)
    if acquire_lock(key, timeout=5):
        try:
            data = db.query(key)
            cache.set(key, data, ex=300)
            return data
        finally:
            release_lock(key)
    else:
        # 3. Couldn't get lock, someone else is fetching
        # Wait briefly, then return stale data or error
        time.sleep(0.5)
        data = cache.get(key)
        if data:
            return data
        raise ServiceUnavailable("Try again")
```

---

## Preventing stampedes in the first place

**Add jitter to TTLs.** Instead of all keys expiring at 5 minutes, use random(4.5min, 5.5min).

**Stagger cache warming.** After deployment, don't let all servers warm their caches simultaneously.

**Monitor hot keys.** Know which keys get the most traffic. Those are your stampede risks.

---

[Next: Real-World Examples →](./10-real-world-examples.md)