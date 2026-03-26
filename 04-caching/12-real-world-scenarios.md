# Real-World Caching Scenarios

[← Back to Caching](./README.md)

---

These are the kinds of problems that actually come up in production and interviews. Not textbook definitions, but "here's a situation, how do you handle it?"

I've included multiple solutions for each because there's rarely one right answer - it depends on your constraints.

---

## 1. The Stampede Problem

**The situation:** You have a cache key that gets 50,000 requests per second. It has a 10-minute TTL. When it expires, all 50,000 requests simultaneously see a cache miss and hit your database.

Your database melts.

**Why this happens:**

```
Time 9:59 - Cache serving 50k req/s, DB load: ~0
Time 10:00 - Key expires
Time 10:00.001 - 50,000 requests all see "cache miss"
Time 10:00.002 - 50,000 queries hit database simultaneously
Time 10:00.003 - Database: "I'm dead"
```

**Solutions:**

**Option 1: Locking**

Only one request fetches from DB. Others wait.

```
Request 1: Cache miss → Acquire lock → Fetch from DB → Update cache → Release lock
Request 2-50000: Cache miss → Lock taken → Wait... → Lock released → Get from cache
```

Downside: Adds latency for waiting requests. Need to handle lock timeouts.

**Option 2: Background refresh**

Don't wait for expiration. Refresh hot keys proactively.

```
Key expires at 10:00
Background job at 9:58: "This key is hot, let me refresh it early"
Key refreshed, new TTL set
Users never see expiration
```

Downside: Need to track which keys are "hot". Extra infrastructure.

**Option 3: Probabilistic early expiration**

As the key approaches expiration, random requests have a chance to refresh it.

```
TTL remaining: 2 min → 5% chance to refresh
TTL remaining: 30 sec → 20% chance to refresh
TTL remaining: 5 sec → 50% chance to refresh
```

One random request refreshes before expiration. No stampede.

Downside: Probabilistic, not guaranteed. But works well in practice.

**Option 4: Request coalescing (singleflight)**

Multiple concurrent requests for the same key get grouped. Only one actually fetches.

```
Time 0ms: Request 1 for "hot_key" → Start fetching
Time 1ms: Request 2 for "hot_key" → "Someone's already fetching, I'll wait for their result"
Time 2ms: Request 3 for "hot_key" → Same, wait
...
Time 100ms: Fetch complete → All 50,000 get the same result
```

This is my favorite for most cases. Libraries like Go's `singleflight` make it easy.

Downside: Only works within a single server. For distributed, you need distributed locking.

**What I'd actually do:** Combine request coalescing (handles the immediate problem) with background refresh for known hot keys (prevents the problem entirely).

---

## 2. "Why don't we just cache everything?"

**The situation:** Someone on your team suggests caching all database queries to improve performance. Sounds reasonable, right?

**Why it's a bad idea:**

**Memory is expensive.**

```
Redis: ~$10-30 per GB per month
SSD: ~$0.10 per GB per month
S3: ~$0.02 per GB per month
```

Redis is 100-500x more expensive than disk. Caching cold data that's rarely accessed is literally burning money.

**Low hit ratio = wasted money.**

If you cache 1000 items but only 10 are accessed frequently, your hit ratio is terrible. 99% of your cache memory is doing nothing.

**More cache = more invalidation complexity.**

Every cached item needs an invalidation strategy. More items = more places where stale data can cause bugs. The Phil Karlton quote exists for a reason.

**Cold start gets worse.**

Server restart → empty cache → ALL requests hit DB. If you cached everything, that's a lot of "everything" that needs to warm up.

**What should be cached:**

| Good candidates | Bad candidates |
|-----------------|----------------|
| Read-heavy data (10:1+ read:write ratio) | Write-heavy data |
| Expensive computations | Cheap queries |
| Frequently accessed (hot data) | Rarely accessed (cold data) |
| Data that can tolerate staleness | Real-time accuracy required |

**The rule I use:** Cache the 20% of data that serves 80% of requests. Monitor hit ratio. If it's below 80%, you're probably caching the wrong things.

---

## 3. Keeping Cache and DB in Sync

**The situation:** You're using cache-aside. Two users update the same record at the same time. How do you prevent the cache from having stale data?

**The problem with "update DB then update cache":**

```
Thread A: Update price to $100
Thread B: Update price to $200

Timeline:
T1: Thread A updates DB to $100
T2: Thread B updates DB to $200
T3: Thread B updates cache to $200
T4: Thread A updates cache to $100  ← WRONG!

Result:
DB: $200 (correct)
Cache: $100 (stale!)
```

Thread A was slower to update cache, so it overwrote Thread B's correct value. Users see wrong price until cache expires.

**The fix: Delete instead of update.**

```
Thread A: Update DB to $100 → DELETE cache key
Thread B: Update DB to $200 → DELETE cache key

Next read: Cache miss → Fetch $200 from DB → Cache $200
```

Delete is idempotent. Doesn't matter which thread deletes first. Next read gets fresh data.

**Other approaches:**

**Versioning:** Include a version number. Only update cache if version is newer.

```
Cache: {price: 100, version: 5}
Update: {price: 200, version: 6}
→ Version 6 > 5, update allowed
```

**Event-driven invalidation:** DB publishes change events. Separate service listens and invalidates cache.

```
DB update → Publish "user:123 changed" → Cache service deletes user:123
```

Decoupled, but adds latency and complexity.

**My recommendation:** Just delete on write. It's simple and handles 95% of cases. Use versioning or events only if you have specific consistency requirements.

---

## 4. Local Cache vs Distributed Cache

**The situation:** Should you use an in-process cache (like Guava/Caffeine) or a distributed cache (like Redis)? Can you use both?

**The tradeoffs:**

| | Local (in-process) | Distributed (Redis) |
|---|---|---|
| Latency | Nanoseconds | ~1 millisecond |
| Capacity | Limited to server RAM | Scales horizontally |
| Consistency | Each server has its own copy | Shared across servers |
| Survives restart | No | Yes (with persistence) |
| Network | None | Required |

**When to use local:**
- Extremely hot data where microseconds matter
- Immutable data (config, feature flags)
- Data that's OK to be inconsistent across servers briefly

**When to use distributed:**
- Shared state (sessions, shopping carts)
- Large datasets
- Need consistency across servers

**Using both (two-tier caching):**

```
Request → L1 (local, nanoseconds) → L2 (Redis, milliseconds) → DB
```

L1: Small, short TTL (30 seconds), hottest data
L2: Larger, longer TTL (5 minutes), shared data

The trick is keeping L1 TTL short so inconsistency window is small. When data changes, invalidate L2 (Redis), and L1 will naturally expire.

**What I'd do:** Start with just Redis. Add local cache only if you have specific hot keys where the 1ms Redis latency is a problem. Premature optimization otherwise.

---

## 5. Cold Start After Deployment

**The situation:** You deploy a new version of your service. The local cache is empty. Suddenly all requests hit the database and latency spikes.

**Why it matters:**

```
Before restart: 95% hit ratio, 5ms average latency
After restart: 0% hit ratio, 500ms average latency

If you have 10 servers and restart them all at once... database goes down.
```

**Solutions:**

**Cache warming:** Before accepting traffic, pre-load hot keys.

```python
def warm_cache():
    hot_keys = get_top_1000_accessed_keys()  # from logs or analytics
    for key in hot_keys:
        data = db.fetch(key)
        cache.set(key, data)
    
    # Now start accepting traffic
```

**Gradual traffic shift:** Don't send 100% traffic to a cold server.

```
T=0: New server gets 1% traffic
T=5min: 10% traffic (cache warming from real requests)
T=10min: 50% traffic
T=15min: 100% traffic
```

Your load balancer can handle this. Kubernetes has readiness probes for similar purposes.

**Rolling restarts:** Never restart all servers at once. Restart one at a time, let it warm up, then restart the next.

**Fallback to L2:** If using two-tier caching, cold L1 falls back to warm L2 (Redis). Impact is reduced.

**The key insight:** Never go from 0 to 100% traffic on a cold cache. Either pre-warm it or ramp up gradually.

---

## 6. Your Redis Bill is $20k/month

**The situation:** Redis cluster is at 90% memory utilization. Finance is asking why the bill is so high. How do you optimize without just adding more nodes?

**Step 1: Figure out what's actually in there.**

```bash
redis-cli --bigkeys  # Find largest keys
redis-cli memory doctor  # Get memory analysis
```

Often 20% of keys use 80% of memory. Find those.

**Step 2: Compress values.**

```python
# Before: 1KB JSON
{"user_id": 12345, "user_name": "John Smith", ...}

# After: Compressed, ~200 bytes
gzip(json.dumps(data))
```

50-80% savings for text data. Worth the CPU tradeoff.

**Step 3: Shorten keys.**

```
# Before
user_profile_data:12345:full_details

# After  
u:12345:p

# Savings: 30+ bytes per key × millions of keys = significant
```

**Step 4: Optimize TTLs.**

Analyze access patterns. If 80% of keys are never accessed after 1 hour, why have a 24-hour TTL?

```
Current: All keys have 24h TTL
Analysis: 80% of keys never accessed after 1h
Fix: Reduce TTL to 1h for cold data
Result: 80% less memory
```

**Step 5: Use efficient data structures.**

```
# Bad: Million separate keys
user:1:name = "John"
user:1:email = "john@example.com"
user:2:name = "Jane"
...

# Good: Hash
HSET user:1 name "John" email "john@example.com"

# 10x less memory overhead
```

**Step 6: Stop caching things that don't need caching.**

Find keys that are written but never read. Find keys with very low hit ratios. Stop caching them.

**Expected result:** 40-60% memory reduction without impacting performance. That's $8-12k/month saved.

---

## 7. Users Keep Requesting IDs That Don't Exist

**The situation:** Your cache-aside logic checks cache, misses, queries DB, finds nothing, returns 404. Next request for same non-existent ID does the same thing. Every request for non-existent data hits your database.

An attacker figures this out and sends 10,000 requests per second for random non-existent IDs. Your database dies.

**This is called cache penetration.**

```
Request: GET /user/999999999 (doesn't exist)

1. Check cache → miss
2. Query DB → null
3. Return 404
4. Nothing cached

Next request for same ID:
1. Check cache → miss (still!)
2. Query DB → null (still!)
3. Return 404

Every request hits DB. Forever.
```

**Solution 1: Cache the null.**

```python
def get_user(user_id):
    cached = cache.get(f"user:{user_id}")
    
    if cached == "NULL":  # Negative cache hit
        return None
    if cached:
        return cached
    
    user = db.get_user(user_id)
    
    if user is None:
        # Cache the absence with SHORT TTL
        cache.set(f"user:{user_id}", "NULL", ttl=300)  # 5 min
        return None
    
    cache.set(f"user:{user_id}", user, ttl=3600)  # 1 hour
    return user
```

Important: Use SHORT TTL for null entries. If the user is created later, you don't want to serve "doesn't exist" for too long.

**Solution 2: Bloom filter.**

A bloom filter can tell you "this ID definitely doesn't exist" without hitting the database.

```
Request for user:999999999
→ Check bloom filter
→ "Definitely not in our system"
→ Return 404 immediately (no DB query)
```

Bloom filters are space-efficient (1 bit per entry) and O(1) lookup. The tradeoff is a small false positive rate (might say "maybe exists" when it doesn't), but zero false negatives.

**Solution 3: Input validation + rate limiting.**

```
Layer 1: Validate input
- User IDs must be positive integers
- User IDs must be < 10 billion
- Reject obviously invalid IDs immediately

Layer 2: Rate limit
- Max 100 requests/second per IP
- Max 10 404s/minute per IP (suspicious pattern)
```

**What I'd do:** Negative caching (simple, handles most cases) + input validation + rate limiting. Add bloom filter only if you're at scale where it matters.

---

## Summary

The common themes across all these scenarios:

1. **There's no single right answer.** Every solution has tradeoffs. Know them.

2. **Think about failure modes.** What happens when cache is down? When it's cold? When it's full?

3. **Measure before optimizing.** Hit ratio, latency percentiles, memory usage. Don't guess.

4. **Start simple.** Cache-aside with TTL handles most cases. Add complexity only when needed.

5. **Delete is safer than update.** For cache invalidation, almost always delete the key rather than updating it.

---

[← Back to Caching](./README.md)