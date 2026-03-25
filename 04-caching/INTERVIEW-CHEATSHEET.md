# 🎯 Caching Interview Cheatsheet

> **Quick review guide for 1-2 days before your interview**

---

## 📝 One-Liner Definitions

| Term | Definition |
|------|------------|
| **Cache** | Fast temporary storage for frequently accessed data |
| **Cache Hit** | Data found in cache |
| **Cache Miss** | Data not in cache, must fetch from source |
| **Hit Ratio** | % of requests served from cache (target: >90%) |
| **TTL** | Time To Live - how long data stays in cache |
| **Eviction** | Removing data when cache is full |

---

## 🏗️ Cache Types (Know These!)

| Type | Where | Speed | Use Case |
|------|-------|-------|----------|
| **Client-side** | Browser/App | Fastest | Static assets, user prefs |
| **CDN** | Edge servers | Very fast | Images, videos, static files |
| **In-memory** | App server RAM | Fast | Hot data, sessions |
| **Distributed** | Redis/Memcached | Fast | Shared state across servers |
| **Database** | DB buffer pool | Medium | Query results |

---

## 📖 5 Caching Strategies (MUST KNOW!)

### 1. Cache-Aside (Lazy Loading) ⭐ Most Common
```
READ:  App → Cache → (miss) → DB → Store in Cache → Return
WRITE: App → DB → Delete from Cache
```
- **Pros**: Simple, cache only has requested data
- **Cons**: First request slow, stale data window
- **Use**: General purpose

### 2. Read-Through
```
READ:  App → Cache (auto-fetches from DB on miss)
```
- Cache handles DB fetching automatically
- **Use**: Read-heavy workloads

### 3. Write-Through
```
WRITE: App → Cache → DB (synchronous)
```
- **Pros**: Cache always has latest data
- **Cons**: Slower writes (2 writes)
- **Use**: Data integrity critical

### 4. Write-Behind (Write-Back)
```
WRITE: App → Cache → (async) → DB
```
- **Pros**: Fastest writes, batching
- **Cons**: Data loss risk if cache fails
- **Use**: Write-heavy, can tolerate loss

### 5. Write-Around
```
WRITE: App → DB (skip cache)
READ:  Normal cache-aside
```
- **Use**: Write-once, read-rarely data

---

## 🔄 Eviction Policies (MUST KNOW!)

| Policy | Evicts | Best For |
|--------|--------|----------|
| **LRU** | Least Recently Used | General purpose ⭐ |
| **LFU** | Least Frequently Used | Stable access patterns |
| **FIFO** | First In, First Out | Simple cases |
| **TTL** | Expired items | Time-sensitive data |
| **Random** | Random item | Unknown patterns |

### LRU Example
```
Cache: [A, B, C] (capacity 3)
Access D → Evict A (least recent) → [B, C, D]
Access B → B moves to end → [C, D, B]
```

---

## 🔴 Redis vs Memcached

| Feature | Redis | Memcached |
|---------|-------|-----------|
| Data types | Rich (lists, sets, hashes) | Strings only |
| Persistence | Yes | No |
| Replication | Yes | No |
| Threading | Single* | Multi |
| Use case | Feature-rich | Simple, fast |

**Choose Redis** for: data structures, persistence, pub/sub
**Choose Memcached** for: simple caching, multi-threaded perf

---

## 🌐 Distributed Caching Concepts

### Consistent Hashing
- **Problem**: Adding/removing servers remaps ALL keys
- **Solution**: Hash ring - only K/N keys remapped
- **Key insight**: Minimizes data movement when scaling

### Replication
- Master-Slave: Writes to master, reads from slaves
- **Benefits**: Read scalability, high availability

### Sharding
- Split data across nodes by key
- **Benefits**: Horizontal scaling, larger cache

---

## ⚠️ Cache Problems & Solutions

### 1. Cache Stampede (Thundering Herd)
**Problem**: Hot key expires → 1000 requests hit DB
**Solutions**:
- Locking (only one fetches)
- Request coalescing
- Background refresh
- Probabilistic early expiration

### 2. Cache Penetration
**Problem**: Requests for non-existent data always hit DB
**Solutions**:
- Cache null results (short TTL)
- Bloom filter

### 3. Cache Avalanche
**Problem**: Many keys expire simultaneously
**Solutions**:
- Add random jitter to TTL
- Stagger cache warming

### 4. Inconsistency Window
**Problem**: Brief period where cache ≠ DB
**Solutions**:
- Accept eventual consistency
- Use write-through
- Versioning

---

## 🎯 Quick Decision Guide

| Scenario | Strategy | Eviction | Tech |
|----------|----------|----------|------|
| General purpose | Cache-Aside | LRU + TTL | Redis |
| Read-heavy | Read-Through | LRU | Redis |
| Write-heavy | Write-Behind | LRU | Redis |
| Data integrity | Write-Through | LRU | Redis |
| Simple caching | Cache-Aside | LRU | Memcached |
| Static content | - | TTL | CDN |
| Sessions | Cache-Aside | TTL | Redis |

---

## 📊 Numbers to Remember

| Metric | Value |
|--------|-------|
| L1 cache latency | ~0.5 ns |
| RAM latency | ~100 ns |
| Redis latency | ~1 ms |
| DB latency | ~10-100 ms |
| Network latency | ~100-200 ms |
| Target hit ratio | >90% |
| Facebook hit ratio | 99%+ |

---

## 🔑 Cache Key Best Practices

**Format**: `{entity}:{id}:{field}`

```
user:123:profile
product:456:price
session:abc123
```

**Rules**:
- Keep keys short
- Make keys predictable
- Include version if needed: `user:123:v2`

---

## ❌ Common Mistakes

1. ❌ Caching everything → Low hit ratio
2. ❌ Very long TTL without invalidation → Stale data
3. ❌ Caching sensitive data → Security risk
4. ❌ Using cache as primary storage → Data loss
5. ❌ Ignoring cache failures → Silent inconsistency

---

## 💡 Interview Tips

1. **Always mention trade-offs** - consistency vs performance
2. **Start with Cache-Aside** - it's the most common
3. **Discuss invalidation** - it's the hardest problem
4. **Know your numbers** - latency comparisons
5. **Real examples** - Twitter fan-out, Netflix EVCache

---

## 🎤 Sample Interview Answer

**Q: "How would you design caching for a product catalog?"**

> "I'd use a **Cache-Aside** strategy with **Redis** as the distributed cache.
> 
> For **reads**: Check Redis first, on miss fetch from DB and cache with TTL.
> 
> For **writes**: Update DB first, then invalidate (delete) the cache key.
> 
> I'd use **LRU eviction** with **5-15 minute TTL** since prices change.
> 
> For **hot products**, I'd use **background refresh** to prevent stampedes.
> 
> To handle **cache failures**, I'd implement a circuit breaker so the system degrades gracefully to DB-only mode.
> 
> Expected **hit ratio**: 90%+, reducing DB load by 10x."

---

*Good luck with your interview! 🚀*

[← Back to Full Guide](./README.md)