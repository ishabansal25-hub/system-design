# 2. Why Caching Matters

[← Back to Caching Index](./README.md) | [← Previous: What is Caching?](./01-what-is-caching.md)

---

## ⚡ Performance Benefits

```
┌────────────────────────────────────────────────────────────────┐
│                    Latency Comparison                           │
├────────────────────────────────────────────────────────────────┤
│                                                                 │
│  L1 Cache Reference          │████                    │ 0.5 ns │
│  L2 Cache Reference          │████████                │ 7 ns   │
│  RAM Reference               │████████████████        │ 100 ns │
│  SSD Read                    │████████████████████████│ 150 μs │
│  HDD Read                    │████████████████████████│ 10 ms  │
│  Network Round Trip          │████████████████████████│ 150 ms │
│                                                                 │
│  Note: 1 ms = 1,000 μs = 1,000,000 ns                          │
│                                                                 │
└────────────────────────────────────────────────────────────────┘
```

### What this means in human terms

| Storage Type | Latency | Human Equivalent |
|--------------|---------|------------------|
| L1 Cache | 0.5 ns | 1 second |
| L2 Cache | 7 ns | 14 seconds |
| RAM | 100 ns | 3.3 minutes |
| SSD | 150 μs | 3.5 days |
| HDD | 10 ms | 8 months |
| Network | 150 ms | 10 years |

**If L1 cache access took 1 second, a network round trip would take 10 years!**

---

## 💰 Cost Benefits

| Metric | Without Cache | With Cache (95% hit rate) |
|--------|---------------|---------------------------|
| DB queries/second | 10,000 | 500 |
| DB replicas needed | 10 | 2 |
| DB cost/month | $10,000 | $2,000 |
| Cache cost/month | $0 | $500 |
| **Total/month** | **$10,000** | **$2,500** |

**75% cost reduction!**

---

## 📊 Real Numbers from Production Systems

| Company | Cache Hit Ratio | Latency Improvement | What They Cache |
|---------|-----------------|---------------------|-----------------|
| Facebook | 99%+ | 100x faster | Social graph, posts, photos |
| Twitter | 99%+ | 50x faster | Tweets, timelines, user data |
| Netflix | 95%+ | 10x faster | Video metadata, recommendations |
| Amazon | 90%+ | 5x faster | Product catalog, cart, sessions |
| Instagram | 99%+ | 100x faster | Photos, stories, user profiles |

---

## 🎯 When to Use Caching

### ✅ Good candidates for caching

| Criteria | Example |
|----------|---------|
| Read-heavy data (read:write ratio > 10:1) | User profiles, product catalog |
| Expensive computations | Complex queries, aggregations |
| Frequently accessed data (hot data) | Homepage content, popular items |
| Data that doesn't change often | Configuration, static content |
| Data that can tolerate slight staleness | News feeds, recommendations |

### ❌ Poor candidates for caching

| Criteria | Example |
|----------|---------|
| Write-heavy data (constantly changing) | Real-time analytics counters |
| Highly personalized data (unique per user) | One-time search results |
| Data requiring real-time accuracy | Stock prices, inventory counts |
| Large objects that rarely repeat | Large file downloads |
| Security-sensitive data | Passwords, authentication tokens |

---

## 📈 Impact Calculator

**Estimate your caching benefit:**

```
Current DB queries/second: Q
Expected hit ratio: H (e.g., 0.90 for 90%)
DB query latency: D (ms)
Cache query latency: C (ms)

New DB queries/second = Q × (1 - H)
Average latency = (H × C) + ((1 - H) × D)

Example:
Q = 1000 queries/sec
H = 0.95 (95% hit ratio)
D = 50ms (database)
C = 1ms (cache)

New DB queries = 1000 × 0.05 = 50/sec (95% reduction!)
Avg latency = (0.95 × 1) + (0.05 × 50) = 0.95 + 2.5 = 3.45ms
(vs 50ms without cache = 14x improvement!)
```

---

[Next: Types of Caches →](./03-types-of-caches.md)