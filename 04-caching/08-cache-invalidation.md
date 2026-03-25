# 8. Cache Invalidation

[← Back to Caching Index](./README.md) | [← Previous: Distributed Caching](./07-distributed-caching.md)

---

## 🎯 The Hardest Problem in Caching

> "There are only two hard things in Computer Science: cache invalidation and naming things."

**Why it's hard:**
- Cache and database can become inconsistent
- Multiple servers may have different cached values
- Race conditions between updates and reads
- Distributed systems make it even harder

---

## 📖 Invalidation Strategies

### 1. TTL-Based Expiration

**How it works**: Data automatically expires after a set time.

```
┌─────────────────────────────────────────────────────────────────┐
│                    TTL-Based Invalidation                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Time 0ms:   SET user:123 {data} TTL=60s                        │
│                                                                  │
│  Time 30s:   GET user:123 → Returns {data} ✓                    │
│                                                                  │
│  Time 60s:   GET user:123 → Returns NULL (expired)              │
│              Cache miss → Fetch fresh data from DB              │
│                                                                  │
│  Pros: Simple, automatic                                         │
│  Cons: Data may be stale until TTL expires                      │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

**Choosing TTL values:**
| Data Type | Suggested TTL | Reason |
|-----------|---------------|--------|
| User sessions | 30 min - 24 hours | Security vs convenience |
| Product catalog | 5-15 minutes | Prices may change |
| Static content | 1 hour - 1 day | Rarely changes |
| Real-time data | 1-5 seconds | Needs freshness |

---

### 2. Event-Based Invalidation

**How it works**: Invalidate cache when data changes.

```
┌─────────────────────────────────────────────────────────────────┐
│                  Event-Based Invalidation                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. User updates profile                                         │
│     ┌─────────┐                                                 │
│     │   App   │                                                 │
│     └────┬────┘                                                 │
│          │                                                       │
│  2. Update database                                              │
│          ▼                                                       │
│     ┌─────────┐                                                 │
│     │   DB    │                                                 │
│     └────┬────┘                                                 │
│          │                                                       │
│  3. Publish "user:123 updated" event                            │
│          ▼                                                       │
│     ┌─────────────┐                                             │
│     │ Message Bus │                                             │
│     │  (Kafka)    │                                             │
│     └──────┬──────┘                                             │
│            │                                                     │
│  4. Cache service receives event → DELETE user:123              │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

**Pros**: Immediate invalidation, no stale data
**Cons**: Complex infrastructure, potential message loss

---

### 3. Version-Based Invalidation

**How it works**: Include version in cache key. When data changes, increment version.

```
┌─────────────────────────────────────────────────────────────────┐
│                 Version-Based Invalidation                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Version 1:                                                      │
│  Cache key: "user:123:v1"                                       │
│  Value: {name: "John", email: "john@old.com"}                   │
│                                                                  │
│  User updates email...                                           │
│                                                                  │
│  Version 2:                                                      │
│  Cache key: "user:123:v2"                                       │
│  Value: {name: "John", email: "john@new.com"}                   │
│                                                                  │
│  Old cache entry (v1) is simply ignored/expires                 │
│  No explicit deletion needed                                     │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

**Pros**: No race conditions, simple rollback
**Cons**: Old versions waste space until they expire

---

## ⚠️ Delete vs Update

**Delete (Recommended):**
1. Update database
2. Delete from cache
3. Next read fetches fresh data

**Update (Risky):**
1. Update database
2. Update cache with new value

**Why delete is better:**
- Simpler (no need to know new value)
- Avoids race conditions
- Cache only stores what's needed

---

[Next: Cache Stampede Prevention →](./09-cache-stampede.md)