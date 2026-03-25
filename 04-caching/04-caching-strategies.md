# 4. Caching Strategies

[← Back to Caching Index](./README.md) | [← Previous: Types of Caches](./03-types-of-caches.md)

---

## 🎯 Overview of Strategies

```
┌─────────────────────────────────────────────────────────────────┐
│                    Caching Strategies Overview                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Strategy          │ Read Path      │ Write Path    │ Use Case  │
│  ─────────────────────────────────────────────────────────────  │
│  Cache-Aside       │ App → Cache    │ App → DB      │ General   │
│  (Lazy Loading)    │      ↓ miss    │ (invalidate)  │ purpose   │
│                    │      DB        │               │           │
│  ─────────────────────────────────────────────────────────────  │
│  Read-Through      │ App → Cache    │ App → DB      │ Read-     │
│                    │ (auto-fetch)   │               │ heavy     │
│  ─────────────────────────────────────────────────────────────  │
│  Write-Through     │ App → Cache    │ App → Cache   │ Data      │
│                    │                │      ↓        │ integrity │
│                    │                │      DB       │           │
│  ─────────────────────────────────────────────────────────────  │
│  Write-Behind      │ App → Cache    │ App → Cache   │ Write-    │
│  (Write-Back)      │                │ (async → DB)  │ heavy     │
│  ─────────────────────────────────────────────────────────────  │
│  Write-Around      │ App → Cache    │ App → DB      │ Write-    │
│                    │      ↓ miss    │ (skip cache)  │ once data │
│                    │      DB        │               │           │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 📖 1. Cache-Aside (Lazy Loading)

**The most common caching strategy!**

### How it works
1. Application checks cache first
2. If cache hit → return data
3. If cache miss → fetch from DB, store in cache, return data
4. On write → update DB, then invalidate (delete) cache

```
┌─────────────────────────────────────────────────────────────────┐
│                    Cache-Aside Pattern                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  READ OPERATION:                                                 │
│                                                                  │
│       ┌─────────┐    1. Check     ┌─────────┐                   │
│       │   App   │ ──────────────► │  Cache  │                   │
│       └────┬────┘                 └────┬────┘                   │
│            │                           │                         │
│            │                      2a. HIT → Return data          │
│            │                           │                         │
│            │                      2b. MISS                       │
│            │                           │                         │
│       3. Fetch    ┌─────────┐          │                        │
│       ◄───────────│   DB    │◄─────────┘                        │
│            │      └─────────┘                                    │
│            │                                                     │
│       4. Store in cache                                          │
│            │                                                     │
│            ▼                                                     │
│       Return data                                                │
│                                                                  │
│  WRITE OPERATION:                                                │
│                                                                  │
│       ┌─────────┐    1. Write     ┌─────────┐                   │
│       │   App   │ ──────────────► │   DB    │                   │
│       └────┬────┘                 └─────────┘                   │
│            │                                                     │
│       2. Invalidate (DELETE from cache)                          │
│            │                                                     │
│            ▼                                                     │
│       ┌─────────┐                                               │
│       │  Cache  │  ← Key deleted                                │
│       └─────────┘                                               │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

**Real-world analogy**: A library with a "recently returned" shelf near the entrance. When you return a book, it goes on this shelf. If someone wants it, they check the shelf first. If not there, they go to the main stacks.

| Pros | Cons |
|------|------|
| ✅ Simple to implement | ❌ First request always slow (cache miss) |
| ✅ Cache only contains requested data | ❌ Potential stale data window |
| ✅ Cache failure doesn't break system | ❌ Application handles caching logic |
| ✅ Works with any database | ❌ Three round trips on miss |

**Best for**: General-purpose caching, read-heavy workloads

---

## 📖 2. Read-Through Cache

### How it works
- Cache sits between application and database
- Cache automatically fetches from DB on miss
- Application only talks to cache (doesn't know about DB)

```
┌─────────────────────────────────────────────────────────────────┐
│                    Read-Through Pattern                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│       ┌─────────┐    1. Request   ┌─────────┐                   │
│       │   App   │ ──────────────► │  Cache  │                   │
│       └─────────┘                 └────┬────┘                   │
│                                        │                         │
│                                   2a. HIT → Return data          │
│                                        │                         │
│                                   2b. MISS                       │
│                                        │                         │
│                                        ▼                         │
│                                   ┌─────────┐                   │
│                                   │   DB    │                   │
│                                   └────┬────┘                   │
│                                        │                         │
│                                   3. Cache fetches data          │
│                                   4. Cache stores it             │
│                                   5. Cache returns to app        │
│                                                                  │
│  Key Difference from Cache-Aside:                               │
│  • Cache handles DB fetching automatically                       │
│  • App doesn't know about DB                                     │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

**Real-world analogy**: A personal assistant who handles all your research. You ask them for information, and they either know it or go find it for you. You never go to the library yourself.

| Pros | Cons |
|------|------|
| ✅ Simpler application code | ❌ First request still slow |
| ✅ Consistent caching logic | ❌ Cache library must support this |
| ✅ Separation of concerns | ❌ Less flexibility |

**Best for**: Read-heavy workloads, when you want simpler application code

---

## 📖 3. Write-Through Cache

### How it works
- Writes go to cache first
- Cache synchronously writes to database
- Data is always consistent between cache and DB

```
┌─────────────────────────────────────────────────────────────────┐
│                    Write-Through Pattern                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│       ┌─────────┐    1. Write     ┌─────────┐                   │
│       │   App   │ ──────────────► │  Cache  │                   │
│       └─────────┘                 └────┬────┘                   │
│                                        │                         │
│                                   2. Store in cache              │
│                                        │                         │
│                                   3. Write to DB (sync)          │
│                                        │                         │
│                                        ▼                         │
│                                   ┌─────────┐                   │
│                                   │   DB    │                   │
│                                   └────┬────┘                   │
│                                        │                         │
│                                   4. DB confirms                 │
│                                        │                         │
│                                   5. Return success to app       │
│                                                                  │
│  Key Point: Write succeeds ONLY if BOTH cache AND DB succeed    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

**Real-world analogy**: A bank teller who updates both the computer system AND the paper ledger for every transaction. The transaction isn't complete until both are updated.

| Pros | Cons |
|------|------|
| ✅ Cache always has latest data | ❌ Higher write latency (2 writes) |
| ✅ No stale data | ❌ Cache may have data never read |
| ✅ Data consistency guaranteed | ❌ Write failures more complex |

**Best for**: Data integrity critical, read-after-write consistency needed

---

## 📖 4. Write-Behind (Write-Back) Cache

### How it works
- Writes go to cache only (immediate return)
- Cache asynchronously writes to database later
- Batches multiple writes for efficiency

```
┌─────────────────────────────────────────────────────────────────┐
│                    Write-Behind Pattern                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│       ┌─────────┐    1. Write     ┌─────────┐                   │
│       │   App   │ ──────────────► │  Cache  │                   │
│       └─────────┘                 └────┬────┘                   │
│            ▲                           │                         │
│            │                      2. Store in cache              │
│            │                           │                         │
│       3. Return success immediately    │                         │
│                                        │                         │
│                                   4. Async write (later)         │
│                                        │                         │
│                                        ▼                         │
│                                   ┌─────────┐                   │
│                                   │   DB    │                   │
│                                   └─────────┘                   │
│                                                                  │
│  Key Point: App gets fast response, DB updated asynchronously   │
│                                                                  │
│  Write Queue:                                                    │
│  ┌─────┬─────┬─────┬─────┬─────┐                               │
│  │ W1  │ W2  │ W3  │ W4  │ W5  │ ──► Batch write to DB         │
│  └─────┴─────┴─────┴─────┴─────┘                               │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

**Real-world analogy**: A restaurant kitchen with order tickets. Orders are written on tickets (cache) and acknowledged immediately. The kitchen processes them in batches when convenient.

| Pros | Cons |
|------|------|
| ✅ Fastest write performance | ❌ Risk of data loss if cache fails |
| ✅ Reduces DB load (batching) | ❌ Complex failure handling |
| ✅ Absorbs write spikes | ❌ Eventual consistency only |

**Best for**: Write-heavy workloads, when some data loss is acceptable

---

## 📖 5. Write-Around Cache

### How it works
- Writes go directly to database (skip cache)
- Cache only populated on reads
- Good for write-once, read-rarely data

```
┌─────────────────────────────────────────────────────────────────┐
│                    Write-Around Pattern                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  WRITE: Goes directly to DB, skips cache                        │
│                                                                  │
│       ┌─────────┐                 ┌─────────┐                   │
│       │   App   │ ──────────────► │   DB    │                   │
│       └─────────┘   Direct write  └─────────┘                   │
│                     (skip cache)                                 │
│                                                                  │
│  READ: Normal cache-aside pattern                                │
│                                                                  │
│       ┌─────────┐    1. Check     ┌─────────┐                   │
│       │   App   │ ──────────────► │  Cache  │                   │
│       └────┬────┘                 └────┬────┘                   │
│            │                           │                         │
│            │                      2a. HIT → Return               │
│            │                      2b. MISS → Fetch from DB       │
│            │                           │                         │
│            ◄───────────────────────────┘                        │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

**Real-world analogy**: A newspaper archive. New articles go directly to the archive (DB). Only when someone requests an old article is it pulled and kept at the front desk (cache) for a while.

| Pros | Cons |
|------|------|
| ✅ Cache not flooded with write-once data | ❌ Cache miss on first read after write |
| ✅ Good for infrequently read data | ❌ Higher read latency for new data |
| ✅ Reduces cache churn | ❌ Not suitable for read-after-write |

**Best for**: Log data, audit trails, write-once data

---

## 📊 Strategy Comparison Summary

| Strategy | Read Latency | Write Latency | Consistency | Complexity | Best Use Case |
|----------|--------------|---------------|-------------|------------|---------------|
| Cache-Aside | Medium (miss) / Fast (hit) | Medium | Eventual | Low | General purpose |
| Read-Through | Medium (miss) / Fast (hit) | Medium | Eventual | Medium | Read-heavy |
| Write-Through | Fast | High | Strong | Medium | Data integrity |
| Write-Behind | Fast | Very Fast | Eventual | High | Write-heavy |
| Write-Around | Medium | Fast | Eventual | Low | Write-once data |

---

## 🎯 Decision Guide

```
                    ┌─────────────────────┐
                    │  What's your main   │
                    │     concern?        │
                    └──────────┬──────────┘
                               │
           ┌───────────────────┼───────────────────┐
           │                   │                   │
           ▼                   ▼                   ▼
    ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
    │   Simple    │     │    Data     │     │   Write     │
    │   & Works   │     │  Integrity  │     │ Performance │
    └──────┬──────┘     └──────┬──────┘     └──────┬──────┘
           │                   │                   │
           ▼                   ▼                   ▼
    ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
    │ Cache-Aside │     │Write-Through│     │Write-Behind │
    └─────────────┘     └─────────────┘     └─────────────┘
```

---

[Next: Cache Eviction Policies →](./05-eviction-policies.md)