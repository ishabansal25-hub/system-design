# 9. Cache Stampede Prevention

[← Back to Caching Index](./README.md) | [← Previous: Cache Invalidation](./08-cache-invalidation.md)

---

## 🐘 What is Cache Stampede?

**Also known as**: Thundering herd, cache avalanche, dog-pile effect

**Scenario:**
1. Popular cache key expires
2. 1000 requests arrive simultaneously
3. All 1000 requests see cache miss
4. All 1000 requests hit database
5. Database overwhelmed → system crash

```
┌─────────────────────────────────────────────────────────────────┐
│                    Cache Stampede Problem                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Normal Operation:                                               │
│  ┌────────┐     ┌───────┐     ┌────┐                           │
│  │Requests│ ──► │ Cache │ ──► │ DB │                           │
│  │  100/s │     │ (HIT) │     │    │  ← 5 queries/s            │
│  └────────┘     └───────┘     └────┘                           │
│                                                                  │
│  Cache Stampede (key expires):                                   │
│  ┌────────┐     ┌───────┐     ┌────┐                           │
│  │Requests│ ──► │ Cache │ ──► │ DB │                           │
│  │  100/s │     │(MISS!)│     │    │  ← 100 queries/s !!!      │
│  └────────┘     └───────┘     └────┘                           │
│                                 │                                │
│                                 ▼                                │
│                            💥 CRASH                              │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 🛡️ Prevention Strategies

### 1. Locking (Mutex)

**How it works**: Only one request fetches from DB; others wait.

```
┌─────────────────────────────────────────────────────────────────┐
│                    Locking Strategy                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Request 1: Cache miss → Acquire lock → Fetch from DB           │
│  Request 2: Cache miss → Lock taken → Wait...                   │
│  Request 3: Cache miss → Lock taken → Wait...                   │
│                                                                  │
│  Request 1: Got data → Store in cache → Release lock            │
│                                                                  │
│  Requests 2,3: Lock released → Get from cache (HIT!)            │
│                                                                  │
│  Result: Only 1 DB query instead of 3                           │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

| Pros | Cons |
|------|------|
| ✅ Prevents stampede | ❌ Adds latency for waiting requests |
| ✅ Guaranteed single fetch | ❌ Potential deadlocks |

---

### 2. Request Coalescing

**How it works**: Multiple requests for same key are grouped; only one fetches.

```
┌─────────────────────────────────────────────────────────────────┐
│                  Request Coalescing                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Time 0ms:   Request 1 for "user:123" → Start fetching          │
│  Time 5ms:   Request 2 for "user:123" → Join Request 1          │
│  Time 10ms:  Request 3 for "user:123" → Join Request 1          │
│  Time 15ms:  Request 4 for "user:456" → Start new fetch         │
│                                                                  │
│  Time 100ms: Request 1 completes                                 │
│              → All joined requests (1,2,3) get same result      │
│                                                                  │
│  Only 2 DB queries for 4 requests!                              │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

**Real-world analogy**: Carpooling. Multiple people going to the same destination share one car.

---

### 3. Probabilistic Early Expiration

**How it works**: Randomly refresh cache before it expires.

```
┌─────────────────────────────────────────────────────────────────┐
│              Probabilistic Early Expiration                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  TTL = 60 seconds                                                │
│                                                                  │
│  Traditional: All requests at T=60s see expiration → Stampede!  │
│                                                                  │
│  Probabilistic:                                                  │
│  • At T=50s: 10% chance to refresh                              │
│  • At T=55s: 30% chance to refresh                              │
│  • At T=58s: 60% chance to refresh                              │
│  • At T=60s: 100% (expired)                                     │
│                                                                  │
│  Result: One random request refreshes early, no stampede!       │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

### 4. Background Refresh

**How it works**: Refresh cache in background before expiration.

```
┌─────────────────────────────────────────────────────────────────┐
│                   Background Refresh                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Cache Entry: user:123                                           │
│  TTL: 60 seconds                                                 │
│  Refresh at: 50 seconds (before expiry)                         │
│                                                                  │
│  Timeline:                                                       │
│  0s        50s                    60s                            │
│  │          │                      │                             │
│  │  Normal  │  Background          │  Would have                 │
│  │  serving │  refresh starts      │  expired here               │
│  │          │  (async)             │                             │
│  │          │                      │                             │
│  │          ▼                      │                             │
│  │     New data fetched            │                             │
│  │     Cache updated               │                             │
│  │     TTL reset to 60s            │                             │
│                                                                  │
│  Users never see cache miss!                                     │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

| Pros | Cons |
|------|------|
| ✅ No user-facing latency | ❌ More complex implementation |
| ✅ Always fresh data | ❌ May refresh unused data |
| ✅ No stampede possible | ❌ Background job management |

---

[Next: Real-World Examples →](./10-real-world-examples.md)