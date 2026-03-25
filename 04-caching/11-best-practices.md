# 11. Best Practices & Common Pitfalls

[← Back to Caching Index](./README.md) | [← Previous: Real-World Examples](./10-real-world-examples.md)

---

## ✅ Best Practices

### Do's

| Practice | Why |
|----------|-----|
| **Use appropriate TTLs** | Balance freshness vs performance |
| **Monitor cache hit ratio** | Should be > 90% for most cases |
| **Implement circuit breakers** | Graceful degradation when cache fails |
| **Use consistent key naming** | `entity:id:field` format (e.g., `user:123:profile`) |
| **Set memory limits** | Prevent cache from consuming all RAM |
| **Plan for cold cache** | System should work (slowly) without cache |
| **Use compression for large values** | Reduce memory usage and network transfer |
| **Implement cache warming** | Pre-populate cache before traffic spikes |

### 🔑 Cache Key Design

**Good key format**: `{entity}:{identifier}:{optional-field}`

| Example | Description |
|---------|-------------|
| `user:123` | User with ID 123 |
| `user:123:profile` | User 123's profile |
| `user:123:orders` | User 123's orders |
| `product:456:price` | Product 456's price |
| `session:abc123` | Session with ID abc123 |

**Key design principles:**
- Keep keys short (memory efficient)
- Make keys predictable (easy to invalidate)
- Include version if needed (`user:123:v2`)
- Avoid special characters

---

## ❌ Common Pitfalls

### Don'ts

| Anti-Pattern | Why It's Bad |
|--------------|--------------|
| **Caching everything** | Wastes memory, low hit ratio |
| **Very long TTLs without invalidation** | Stale data problems |
| **Caching sensitive data** | Security risk |
| **Ignoring cache failures** | Silent data inconsistency |
| **Using cache as primary storage** | Data loss risk |
| **Complex cache keys** | Hard to invalidate, debug |
| **Caching null/empty results** | May hide real issues |

---

## ⚠️ Common Problems

### 1. Cache Penetration
**Problem**: Requests for non-existent data always hit database.

**Solutions:**
- Cache null results (with short TTL)
- Bloom filter (check if key might exist)
- Rate limiting

### 2. Cache Breakdown
**Problem**: Hot key expires, causing sudden DB load.

**Solutions:**
- Never expire hot keys (or use very long TTL)
- Use locking for hot key refresh
- Background refresh before expiration

### 3. Cache Avalanche
**Problem**: Many keys expire at the same time.

**Solutions:**
- Add random jitter to TTL (TTL = 60 + random(0,10))
- Stagger cache warming
- Use different TTLs for different data types

### 4. Inconsistency Window
**Problem**: Brief period where cache and DB are inconsistent.

**Mitigation:**
- Accept eventual consistency
- Use write-through for strong consistency
- Use versioning

### 5. Memory Pressure
**Problem**: Cache grows unbounded, causes OOM.

**Solutions:**
- Set max memory limits
- Use appropriate eviction policy
- Monitor memory usage
- Size cache appropriately for workload

---

## 📊 Quick Reference

| Scenario | Recommended Approach |
|----------|---------------------|
| General purpose | Cache-Aside + LRU + TTL |
| Read-heavy | Read-Through + LRU |
| Write-heavy | Write-Behind + batching |
| Data integrity critical | Write-Through |
| Session storage | Redis with TTL |
| Static content | CDN |
| Hot data | Long TTL + background refresh |

---

## 🎯 Summary

1. **Caching is essential** for building scalable systems
2. **Choose the right strategy** based on your read/write patterns
3. **Cache invalidation is hard** - plan for it carefully
4. **Monitor everything** - hit ratio, latency, memory
5. **Plan for failure** - system should work without cache
6. **Start simple** - Cache-Aside with TTL works for most cases

---

*Happy Caching! 🚀*

[← Back to Caching Index](./README.md)