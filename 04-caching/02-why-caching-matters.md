# Why Caching Matters

[← Back to Caching](./README.md) | [← Previous: What is Caching?](./01-what-is-caching.md)

---

## The speed difference is insane

Here's why caching works so well - the latency gap between memory and everything else is enormous:

| Storage | Latency | If RAM took 1 minute... |
|---------|---------|-------------------------|
| L1 Cache | 0.5 ns | 0.3 seconds |
| L2 Cache | 7 ns | 4 seconds |
| RAM | 100 ns | 1 minute |
| SSD | 150 μs | 25 hours |
| Database query | 10 ms | 70 days |
| Network call | 150 ms | 3 years |

That last row is the important one. A database query that takes 10ms feels instant to humans, but it's 100,000x slower than RAM. When you're handling thousands of requests per second, that adds up fast.

---

## The cost math

Let's say you have an API doing 10,000 requests per second, each hitting the database.

**Without caching:**
- 10,000 DB queries/second
- Need beefy database (or multiple replicas)
- Estimated cost: ~$10,000/month

**With caching (95% hit ratio):**
- 500 DB queries/second (95% served from cache)
- Smaller database works fine
- Redis cluster: ~$500/month
- Total: ~$2,500/month

That's 75% savings. And the system is faster too.

The math is simple: RAM is expensive per GB, but you need way less of it because you're only caching hot data. A $500/month Redis instance can replace $10,000/month of database capacity.

---

## What the big companies see

These numbers get thrown around a lot, but they're real:

- **Facebook**: 99%+ hit ratio on their social graph cache. Without it, they'd need 100x more database capacity.
- **Twitter**: Timeline cache serves 99%+ of requests. The "fail whale" era was partly cache problems.
- **Netflix**: Video metadata cached at 95%+. Recommendations are pre-computed and cached.

The pattern: anything that gets read way more than it gets written is a caching candidate.

---

## When caching makes sense

**Good candidates:**

- Read-heavy data (10:1 read:write ratio or higher)
- Expensive queries (joins, aggregations, anything that takes >10ms)
- Hot data (the 20% of data that serves 80% of requests)
- Data that can be slightly stale (user profiles, product info, feeds)

**Bad candidates:**

- Write-heavy data (you'll spend all your time invalidating)
- Unique requests (search results for queries nobody else will make)
- Real-time accuracy required (stock prices, inventory during checkout)
- Security-sensitive data (don't cache passwords or tokens)

---

## Quick sanity check

Before adding caching, ask:

1. **Is this data read more than written?** If it changes every request, caching won't help.

2. **Is this data accessed frequently?** Caching data that's requested once a day is pointless.

3. **Can users tolerate stale data?** If showing a 5-minute-old price is unacceptable, caching gets complicated.

4. **Is the source slow or expensive?** If your query already takes 1ms, caching saves you almost nothing.

If you answered "yes" to most of these, caching will probably help. If not, you might be adding complexity for minimal gain.

---

## Back-of-napkin calculation

Want to estimate if caching is worth it for your use case?

```
Current queries/second: 1000
Expected hit ratio: 90%
DB latency: 50ms
Cache latency: 1ms

After caching:
- DB queries drop to: 1000 × 10% = 100/sec
- Average latency: (90% × 1ms) + (10% × 50ms) = 0.9 + 5 = 5.9ms

Before: 50ms average, 1000 DB queries/sec
After: 5.9ms average, 100 DB queries/sec
```

That's 8x faster and 90% less database load. Usually worth the added complexity.

---

[Next: Types of Caches →](./03-types-of-caches.md)