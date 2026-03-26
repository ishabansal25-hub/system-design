# Best Practices and Mistakes to Avoid

[← Back to Caching](./README.md) | [← Previous: Real-World Examples](./10-real-world-examples.md)

---

After going through all the concepts, here's the practical stuff - what to do, what not to do, and the problems you'll actually run into.

---

## Things that work well

**Use consistent key naming.**

Pick a format and stick with it:
```
{entity}:{id}:{field}

user:123:profile
user:123:orders
product:456:price
session:abc123
```

Makes it easy to find keys, invalidate them, and debug issues.

**Set TTLs on everything.**

Even if you have explicit invalidation, TTL is your safety net. If invalidation fails, data eventually expires anyway.

**Monitor your hit ratio.**

If it's below 80%, something's wrong:
- You're caching the wrong things
- TTLs are too short
- Cache is too small
- Access patterns are too random

**Plan for cache failure.**

Your app should work without the cache. Slower, yes. But not broken. Test this by killing Redis and seeing what happens.

**Delete on write, not update.**

When data changes:
```python
# Good
db.update(user_id, data)
cache.delete(f"user:{user_id}")

# Bad (race conditions)
db.update(user_id, data)
cache.set(f"user:{user_id}", data)
```

**Compress large values.**

If you're caching JSON blobs over 1KB, compression can save 50-80% memory. Worth the CPU tradeoff.

**Add jitter to TTLs.**

Instead of `TTL=300`, use `TTL=random(270, 330)`. Prevents synchronized expirations that cause stampedes.

---

## Common mistakes

**Caching everything.**

"Let's just cache all database queries!" No. Cache hot data with high read:write ratios. Caching cold data wastes memory and gives you a terrible hit ratio.

**Very long TTLs without invalidation.**

Setting TTL=24h and hoping for the best. When data changes, users see stale info for up to a day. Either shorten TTL or add explicit invalidation.

**Caching sensitive data.**

Passwords, tokens, PII in cache. If your cache is compromised, that's a security incident. Be careful what you cache.

**Using cache as primary storage.**

Cache is temporary. It can be evicted, it can fail, it can restart empty. Your source of truth is the database.

**Ignoring cache failures.**

```python
# Bad
data = cache.get(key)  # What if this throws?
return data

# Better
try:
    data = cache.get(key)
    if data:
        return data
except CacheError:
    log.warning("Cache unavailable")
# Fall through to database
```

**Complex cache keys.**

```
# Bad
f"user_{user_id}_profile_data_v2_region_{region}_lang_{lang}"

# Better
f"user:{user_id}:profile"
```

Keep keys simple. Complex keys are hard to invalidate and debug.

---

## Problems you'll encounter

**Cache penetration** - Requests for non-existent data bypass cache and hit DB every time.

Fix: Cache null results with short TTL, or use a bloom filter.

**Cache stampede** - Hot key expires, thousands of requests rebuild it simultaneously.

Fix: Locking, request coalescing, or background refresh.

**Cache avalanche** - Many keys expire at the same time, overwhelming the database.

Fix: Add random jitter to TTLs. Stagger cache warming.

**Inconsistency window** - Brief period where cache and DB disagree.

Fix: Accept it (eventual consistency), or use write-through for strong consistency.

**Memory pressure** - Cache grows until it OOMs.

Fix: Set max memory limits. Use appropriate eviction policy. Monitor usage.

---

## Quick decision guide

| Situation | What to do |
|-----------|------------|
| Starting out | Cache-Aside + LRU + TTL |
| Read-heavy API | Read-Through or Cache-Aside |
| Write-heavy | Write-Behind (if you can tolerate data loss) |
| Need strong consistency | Write-Through |
| User sessions | Redis with TTL |
| Static assets | CDN |
| Hot keys | Long TTL + background refresh |
| Unknown access patterns | Start with short TTL, measure, adjust |

---

## Monitoring checklist

Track these metrics:

- **Hit ratio** - Should be >80%, ideally >90%
- **Latency** - p50, p95, p99 for cache operations
- **Memory usage** - Are you approaching limits?
- **Eviction rate** - High eviction = cache too small
- **Connection count** - Are you leaking connections?
- **Error rate** - Cache failures, timeouts

Set alerts for:
- Hit ratio drops below threshold
- Latency spikes
- Memory approaching limit
- Error rate increases

---

## Final thoughts

Caching isn't complicated in concept, but it's easy to mess up in practice. The most common issues I've seen:

1. **Over-engineering.** Start simple. Cache-Aside with TTL handles most cases.

2. **Under-monitoring.** You can't fix what you can't see. Measure hit ratio from day one.

3. **Forgetting invalidation.** Adding caching is easy. Keeping cache and DB in sync is the hard part.

4. **Not testing failure modes.** What happens when Redis is down? When it's slow? When it's full? Test these scenarios.

5. **Premature optimization.** Don't add caching until you have a performance problem. Then add it where it matters most.

---

[← Back to Caching](./README.md)