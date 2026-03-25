# 🚀 Caching - Complete Guide

> **"There are only two hard things in Computer Science: cache invalidation and naming things."** — Phil Karlton

This comprehensive guide covers everything you need to know about caching, from basic concepts to advanced distributed caching strategies. Whether you're a junior engineer learning the fundamentals or a senior engineer designing large-scale systems, this guide has something for you.

---

## 📚 Table of Contents

1. [What is Caching?](#1-what-is-caching)
2. [Why Caching Matters](#2-why-caching-matters)
3. [Types of Caches](#3-types-of-caches)
4. [Caching Strategies](#4-caching-strategies)
5. [Cache Eviction Policies](#5-cache-eviction-policies)
6. [Caching Technologies](#6-caching-technologies)
7. [Distributed Caching](#7-distributed-caching)
8. [Cache Invalidation](#8-cache-invalidation)
9. [Cache Stampede Prevention](#9-cache-stampede-prevention)
10. [Real-World Examples](#10-real-world-examples)
11. [Best Practices](#11-best-practices)
12. [Common Pitfalls](#12-common-pitfalls)

---

## 1. What is Caching?

### 🎯 Simple Explanation (For Beginners)

Imagine you're a librarian. Every time someone asks for a popular book, you have to walk to the back of the library, find the book, and bring it to them. This takes time!

**Smart solution**: Keep the most popular books on your desk. When someone asks for them, you can hand them over immediately without walking anywhere.

**That's caching!** 📚

In computing terms:
- **Cache** = Your desk (fast, limited space)
- **Database/Disk** = Back of the library (slow, lots of space)
- **Popular books** = Frequently accessed data

### 📖 Technical Definition

A **cache** is a high-speed data storage layer that stores a subset of data, typically transient in nature, so that future requests for that data are served faster than accessing the data's primary storage location.

```
┌─────────────────────────────────────────────────────────────────┐
│                        Without Cache                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   User Request ──────────────────────────────► Database          │
│        │                                           │             │
│        │         (Every request hits DB)           │             │
│        │                                           │             │
│        ◄───────────────────────────────────────────┘             │
│                     Response (Slow: ~100ms)                      │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                         With Cache                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   User Request ────► Cache ─── HIT ───► Response (Fast: ~1ms)   │
│                        │                                         │
│                       MISS                                       │
│                        │                                         │
│                        ▼                                         │
│                    Database                                      │
│                        │                                         │
│                        ▼                                         │
│              Store in Cache + Response                           │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 🔑 Key Terminology

| Term | Definition | Example |
|------|------------|---------|
| **Cache Hit** | Data found in cache | User profile found in Redis |
| **Cache Miss** | Data NOT in cache, must fetch from source | User profile not in cache, fetch from DB |
| **Hit Ratio** | % of requests served from cache | 95% hit ratio = 95 out of 100 requests from cache |
| **TTL (Time To Live)** | How long data stays in cache | TTL=300s means data expires after 5 minutes |
| **Eviction** | Removing data from cache | Removing old data when cache is full |
| **Warm Cache** | Cache populated with data | After system runs for a while |
| **Cold Cache** | Empty or newly started cache | Right after system restart |
| **Cache Warming** | Pre-populating cache with data | Loading popular items before traffic spike |
| **Stale Data** | Outdated data in cache | Cache has old price, DB has new price |

### 🧠 The Principle of Locality

Caching works because of two fundamental principles:

#### 1. Temporal Locality
**"If you accessed something recently, you'll likely access it again soon."**

Real-world example: 
- You just looked up a word in the dictionary
- You'll probably look at that same page again in the next few minutes
- Keep that page bookmarked!

System example:
- User views their profile → likely to view it again
- Cache the profile for quick access

#### 2. Spatial Locality
**"If you accessed something, you'll likely access nearby things."**

Real-world example:
- You're reading page 50 of a book
- You'll probably read pages 51, 52, 53 next
- Pre-load nearby pages!

System example:
- User views product #100 → might view related products
- Cache related products together

---

## 2. Why Caching Matters

### ⚡ Performance Benefits

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

**What this means in human terms:**

| Storage Type | Latency | Human Equivalent |
|--------------|---------|------------------|
| L1 Cache | 0.5 ns | 1 second |
| L2 Cache | 7 ns | 14 seconds |
| RAM | 100 ns | 3.3 minutes |
| SSD | 150 μs | 3.5 days |
| HDD | 10 ms | 8 months |
| Network | 150 ms | 10 years |

If L1 cache access took 1 second, a network round trip would take **10 years**!

### 💰 Cost Benefits

| Metric | Without Cache | With Cache (95% hit rate) |
|--------|---------------|---------------------------|
| DB queries/second | 10,000 | 500 |
| DB replicas needed | 10 | 2 |
| DB cost/month | $10,000 | $2,000 |
| Cache cost/month | $0 | $500 |
| **Total/month** | **$10,000** | **$2,500** |

**75% cost reduction!**

### 📊 Real Numbers from Production Systems

| Company | Cache Hit Ratio | Latency Improvement | What They Cache |
|---------|-----------------|---------------------|-----------------|
| Facebook | 99%+ | 100x faster | Social graph, posts, photos |
| Twitter | 99%+ | 50x faster | Tweets, timelines, user data |
| Netflix | 95%+ | 10x faster | Video metadata, recommendations |
| Amazon | 90%+ | 5x faster | Product catalog, cart, sessions |
| Instagram | 99%+ | 100x faster | Photos, stories, user profiles |

### 🎯 When to Use Caching

**Good candidates for caching:**
- ✅ Read-heavy data (read:write ratio > 10:1)
- ✅ Expensive computations (complex queries, aggregations)
- ✅ Frequently accessed data (hot data)
- ✅ Data that doesn't change often
- ✅ Data that can tolerate slight staleness

**Poor candidates for caching:**
- ❌ Write-heavy data (constantly changing)
- ❌ Highly personalized data (unique per user)
- ❌ Data requiring real-time accuracy (stock prices, inventory)
- ❌ Large objects that rarely repeat
- ❌ Security-sensitive data (passwords, tokens)

---

## 3. Types of Caches

### 🏗️ Cache Hierarchy

```
┌─────────────────────────────────────────────────────────────────┐
│                      Cache Hierarchy                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                    CLIENT SIDE                           │    │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐      │    │
│  │  │   Browser   │  │   Mobile    │  │   Desktop   │      │    │
│  │  │    Cache    │  │  App Cache  │  │  App Cache  │      │    │
│  │  └─────────────┘  └─────────────┘  └─────────────┘      │    │
│  └─────────────────────────────────────────────────────────┘    │
│                              │                                   │
│                              ▼                                   │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                       CDN LAYER                          │    │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐      │    │
│  │  │  CloudFront │  │  Cloudflare │  │   Akamai    │      │    │
│  │  │   (Edge)    │  │   (Edge)    │  │   (Edge)    │      │    │
│  │  └─────────────┘  └─────────────┘  └─────────────┘      │    │
│  └─────────────────────────────────────────────────────────┘    │
│                              │                                   │
│                              ▼                                   │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                   APPLICATION LAYER                      │    │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐      │    │
│  │  │  In-Memory  │  │   Local     │  │  Distributed│      │    │
│  │  │   (Dict)    │  │   (File)    │  │   (Redis)   │      │    │
│  │  └─────────────┘  └─────────────┘  └─────────────┘      │    │
│  └─────────────────────────────────────────────────────────┘    │
│                              │                                   │
│                              ▼                                   │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                    DATABASE LAYER                        │    │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐      │    │
│  │  │   Query     │  │   Buffer    │  │  Connection │      │    │
│  │  │   Cache     │  │    Pool     │  │    Pool     │      │    │
│  │  └─────────────┘  └─────────────┘  └─────────────┘      │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 📱 1. Client-Side Cache

**What**: Data stored on user's device (browser, mobile app)

**Examples**:
- Browser cache (images, CSS, JS files)
- LocalStorage / SessionStorage
- Mobile app SQLite cache
- Service Workers (PWA offline cache)

**Characteristics**:
| Aspect | Details |
|--------|---------|
| Location | User's device |
| Control | Limited (user can clear) |
| Size | ~5-50 MB typical |
| Speed | Fastest (no network) |
| Scope | Single user only |

**Real-world analogy**: 
Your phone's photo gallery. Photos are stored locally so you can view them instantly without downloading again.

### 🌐 2. CDN Cache (Content Delivery Network)

**What**: Geographically distributed servers that cache static content close to users

```
┌─────────────────────────────────────────────────────────────────┐
│                     CDN Architecture                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│     User in India          User in USA          User in Europe  │
│          │                      │                      │        │
│          ▼                      ▼                      ▼        │
│    ┌──────────┐           ┌──────────┐           ┌──────────┐   │
│    │  Mumbai  │           │   NYC    │           │  London  │   │
│    │  Edge    │           │  Edge    │           │  Edge    │   │
│    │  Server  │           │  Server  │           │  Server  │   │
│    └────┬─────┘           └────┬─────┘           └────┬─────┘   │
│         │                      │                      │         │
│         │    Cache Miss?       │    Cache Miss?       │         │
│         │                      │                      │         │
│         └──────────────────────┼──────────────────────┘         │
│                                │                                 │
│                                ▼                                 │
│                    ┌─────────────────────┐                      │
│                    │    Origin Server    │                      │
│                    │   (Your Backend)    │                      │
│                    └─────────────────────┘                      │
│                                                                  │
│  Benefits:                                                       │
│  • User in India: 20ms latency (from Mumbai edge)               │
│  • Without CDN: 200ms latency (from US origin)                  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

**What CDNs cache**:
- Static files (images, videos, CSS, JS)
- API responses (with proper headers)
- HTML pages (for static sites)

**Popular CDN providers**:
| Provider | Strengths |
|----------|-----------|
| CloudFront | AWS integration, Lambda@Edge |
| Cloudflare | DDoS protection, Workers |
| Akamai | Enterprise, largest network |
| Fastly | Real-time purging, edge compute |

**Real-world analogy**: 
Amazon warehouses. Instead of shipping everything from one central warehouse, they have fulfillment centers near major cities for faster delivery.

### 🖥️ 3. Application-Level Cache

**What**: Cache within your application servers

#### a) In-Memory Cache (Single Server)

**What**: Data stored in application's RAM

**Characteristics**:
| Aspect | Details |
|--------|---------|
| Speed | Extremely fast (nanoseconds) |
| Scope | Single server only |
| Persistence | Lost on restart |
| Size | Limited by server RAM |

**When to use**:
- Small datasets that fit in memory
- Single-server applications
- Temporary computation results
- Session data (with sticky sessions)

**Real-world analogy**: 
Your brain's short-term memory. Quick to access but limited capacity and forgotten when you sleep.

#### b) Distributed Cache (Multiple Servers)

**What**: Shared cache accessible by all application servers

```
┌─────────────────────────────────────────────────────────────────┐
│                    Distributed Cache                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐           │
│   │  App    │  │  App    │  │  App    │  │  App    │           │
│   │Server 1 │  │Server 2 │  │Server 3 │  │Server 4 │           │
│   └────┬────┘  └────┬────┘  └────┬────┘  └────┬────┘           │
│        │            │            │            │                  │
│        └────────────┴─────┬──────┴────────────┘                  │
│                           │                                      │
│                           ▼                                      │
│              ┌─────────────────────────┐                        │
│              │    Distributed Cache    │                        │
│              │   (Redis / Memcached)   │                        │
│              │                         │                        │
│              │  ┌─────┐ ┌─────┐ ┌─────┐│                        │
│              │  │Node1│ │Node2│ │Node3││                        │
│              │  └─────┘ └─────┘ └─────┘│                        │
│              └─────────────────────────┘                        │
│                                                                  │
│  Benefits:                                                       │
│  • All app servers share same cache                             │
│  • Survives individual server failures                          │
│  • Scales horizontally                                          │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

**Characteristics**:
| Aspect | Details |
|--------|---------|
| Speed | Fast (sub-millisecond) |
| Scope | All servers in cluster |
| Persistence | Configurable |
| Size | Scales with nodes |

**Real-world analogy**: 
A shared Google Doc. Everyone on the team can access and update it, and changes are visible to all.

### 🗄️ 4. Database Cache

**What**: Caching at the database level

**Types**:

| Type | Description | Example |
|------|-------------|---------|
| Query Cache | Caches query results | Same SELECT returns cached result |
| Buffer Pool | Caches data pages in memory | Frequently accessed rows stay in RAM |
| Connection Pool | Reuses database connections | Avoids connection overhead |
| Materialized Views | Pre-computed query results | Complex aggregations stored as tables |

**Real-world analogy**: 
A restaurant's prep station. Commonly used ingredients are pre-chopped and ready, rather than preparing everything from scratch for each order.

---

## 4. Caching Strategies

### 🎯 Overview of Strategies

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

### 📖 1. Cache-Aside (Lazy Loading)

**The most common caching strategy!**

**How it works**:
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

**Real-world analogy**: 
A library with a "recently returned" shelf near the entrance. When you return a book, it goes on this shelf. If someone wants it, they check the shelf first. If not there, they go to the main stacks.

| Pros | Cons |
|------|------|
| ✅ Simple to implement | ❌ First request always slow (cache miss) |
| ✅ Cache only contains requested data | ❌ Potential stale data window |
| ✅ Cache failure doesn't break system | ❌ Application handles caching logic |
| ✅ Works with any database | ❌ Three round trips on miss |

**Best for**: General-purpose caching, read-heavy workloads

---

### 📖 2. Read-Through Cache

**How it works**:
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

**Real-world analogy**: 
A personal assistant who handles all your research. You ask them for information, and they either know it or go find it for you. You never go to the library yourself.

| Pros | Cons |
|------|------|
| ✅ Simpler application code | ❌ First request still slow |
| ✅ Consistent caching logic | ❌ Cache library must support this |
| ✅ Separation of concerns | ❌ Less flexibility |

**Best for**: Read-heavy workloads, when you want simpler application code

---

### 📖 3. Write-Through Cache

**How it works**:
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

**Real-world analogy**: 
A bank teller who updates both the computer system AND the paper ledger for every transaction. The transaction isn't complete until both are updated.

| Pros | Cons |
|------|------|
| ✅ Cache always has latest data | ❌ Higher write latency (2 writes) |
| ✅ No stale data | ❌ Cache may have data never read |
| ✅ Data consistency guaranteed | ❌ Write failures more complex |

**Best for**: Data integrity critical, read-after-write consistency needed

---

### 📖 4. Write-Behind (Write-Back) Cache

**How it works**:
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

**Real-world analogy**: 
A restaurant kitchen with order tickets. Orders are written on tickets (cache) and acknowledged immediately. The kitchen processes them in batches when convenient.

| Pros | Cons |
|------|------|
| ✅ Fastest write performance | ❌ Risk of data loss if cache fails |
| ✅ Reduces DB load (batching) | ❌ Complex failure handling |
| ✅ Absorbs write spikes | ❌ Eventual consistency only |

**Best for**: Write-heavy workloads, when some data loss is acceptable

---

### 📖 5. Write-Around Cache

**How it works**:
- Writes go directly to database (skip cache)
- Cache only populated on reads
- Good for write-once, read-rarely data

```
┌─────────────────────────────────────────────────────────────────┐
│                    Write-Around Pattern                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  WRITE OPERATION:                                                │
│                                                                  │
│       ┌─────────┐                 ┌─────────┐                   │
│       │   App   │ ──────────────► │   DB    │                   │
│       └─────────┘   Direct write  └─────────┘                   │
│                     (skip cache)                                 │
│                                                                  │
│  READ OPERATION:                                                 │
│                                                                  │
│       ┌─────────┐    1. Check     ┌─────────┐                   │
│       │   App   │ ──────────────► │  Cache  │                   │
│       └────┬────┘                 └────┬────┘                   │
│            │                           │                         │
│            │                      2a. HIT → Return               │
│            │                           │                         │
│            │                      2b. MISS → Fetch from DB       │
│            │                           │                         │
│            ◄───────────────────────────┘                        │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

**Real-world analogy**: 
A newspaper archive. New articles go directly to the archive (DB). Only when someone requests an old article is it pulled and kept at the front desk (cache) for a while.

| Pros | Cons |
|------|------|
| ✅ Cache not flooded with write-once data | ❌ Cache miss on first read after write |
| ✅ Good for infrequently read data | ❌ Higher read latency for new data |
| ✅ Reduces cache churn | ❌ Not suitable for read-after-write |

**Best for**: Log data, audit trails, write-once data

---

### 📊 Strategy Comparison Summary

| Strategy | Read Latency | Write Latency | Consistency | Complexity | Best Use Case |
|----------|--------------|---------------|-------------|------------|---------------|
| Cache-Aside | Medium (miss) / Fast (hit) | Medium | Eventual | Low | General purpose |
| Read-Through | Medium (miss) / Fast (hit) | Medium | Eventual | Medium | Read-heavy |
| Write-Through | Fast | High | Strong | Medium | Data integrity |
| Write-Behind | Fast | Very Fast | Eventual | High | Write-heavy |
| Write-Around | Medium | Fast | Eventual | Low | Write-once data |

---

## 5. Cache Eviction Policies

### 🎯 Why Eviction is Needed

Cache has limited memory. When it's full, we must remove something to add new data. The eviction policy decides **what to remove**.

```
┌─────────────────────────────────────────────────────────────────┐
│                    Cache Eviction Scenario                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Cache is FULL (capacity: 5 items)                              │
│  ┌─────┬─────┬─────┬─────┬─────┐                               │
│  │  A  │  B  │  C  │  D  │  E  │                               │
│  └─────┴─────┴─────┴─────┴─────┘                               │
│                                                                  │
│  New item F arrives...                                           │
│                                                                  │
│  Question: Which item should be removed?                         │
│  • A (oldest)?                                                   │
│  • B (least recently used)?                                      │
│  • C (least frequently used)?                                    │
│  • Random?                                                       │
│                                                                  │
│  Answer depends on the EVICTION POLICY                          │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 📖 Common Eviction Policies

#### 1. LRU (Least Recently Used)

**Concept**: Remove the item that hasn't been accessed for the longest time.

**Assumption**: If you haven't used it recently, you probably won't need it soon.

```
┌─────────────────────────────────────────────────────────────────┐
│                         LRU Example                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Cache capacity: 3                                               │
│                                                                  │
│  Access sequence: A, B, C, A, D                                  │
│                                                                  │
│  Step 1: Access A    [A]           (A is most recent)           │
│  Step 2: Access B    [A, B]        (B is most recent)           │
│  Step 3: Access C    [A, B, C]     (C is most recent) - FULL    │
│  Step 4: Access A    [B, C, A]     (A moves to most recent)     │
│  Step 5: Access D    [C, A, D]     (B evicted - least recent)   │
│                       ↑                                          │
│                       B was removed because it was               │
│                       least recently used                        │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

**Real-world analogy**: 
Your browser's "recently visited" list. Sites you haven't visited in a while drop off the list.

| Pros | Cons |
|------|------|
| ✅ Simple to understand | ❌ Doesn't consider frequency |
| ✅ Works well for most cases | ❌ One-time access can evict useful data |
| ✅ O(1) with proper implementation | ❌ Scan resistance issues |

**Best for**: General-purpose caching, web applications

---

#### 2. LFU (Least Frequently Used)

**Concept**: Remove the item that has been accessed the fewest times.

**Assumption**: Items accessed many times are more valuable than items accessed few times.

```
┌─────────────────────────────────────────────────────────────────┐
│                         LFU Example                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Cache capacity: 3                                               │
│                                                                  │
│  Access sequence: A, A, A, B, B, C, D                            │
│                                                                  │
│  After A, A, A:   [A(3)]                                        │
│  After B, B:      [A(3), B(2)]                                  │
│  After C:         [A(3), B(2), C(1)]    - FULL                  │
│  After D:         [A(3), B(2), D(1)]    - C evicted (freq=1)    │
│                                                                  │
│  Item    │ Frequency                                             │
│  ────────┼──────────                                             │
│  A       │ 3 (keep)                                              │
│  B       │ 2 (keep)                                              │
│  C       │ 1 (evicted)                                           │
│  D       │ 1 (new)                                               │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

**Real-world analogy**: 
A store deciding which products to keep on shelves. Products that sell frequently stay; rarely-sold items are removed.

| Pros | Cons |
|------|------|
| ✅ Keeps frequently used items | ❌ New items may be evicted quickly |
| ✅ Good for stable access patterns | ❌ Old popular items may never be evicted |
| ✅ Considers historical usage | ❌ More complex to implement |

**Best for**: Stable workloads, when access patterns don't change much

---

#### 3. FIFO (First In, First Out)

**Concept**: Remove the oldest item (first one added).

**Assumption**: Older items are less relevant than newer items.

```
┌─────────────────────────────────────────────────────────────────┐
│                        FIFO Example                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Cache capacity: 3                                               │
│                                                                  │
│  Access sequence: A, B, C, D                                     │
│                                                                  │
│  Step 1: Add A     [A]                                          │
│  Step 2: Add B     [A, B]                                       │
│  Step 3: Add C     [A, B, C]     - FULL                         │
│  Step 4: Add D     [B, C, D]     - A evicted (first in)         │
│                     ↑                                            │
│                     A was removed because it was                 │
│                     the first item added                         │
│                                                                  │
│  Note: Even if A was accessed 100 times, it still gets evicted  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

**Real-world analogy**: 
A queue at a coffee shop. First person in line gets served first and leaves first.

| Pros | Cons |
|------|------|
| ✅ Very simple to implement | ❌ Ignores access patterns |
| ✅ Predictable behavior | ❌ May evict frequently used items |
| ✅ Low overhead | ❌ Poor hit ratio in most cases |

**Best for**: Simple scenarios, when all items have similar access patterns

---

#### 4. TTL (Time To Live)

**Concept**: Each item has an expiration time. Remove when expired.

**Assumption**: Data becomes stale after a certain time.

```
┌─────────────────────────────────────────────────────────────────┐
│                         TTL Example                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Time: 0s                                                        │
│  ┌─────────────┬─────────────┬─────────────┐                    │
│  │ A (TTL=60s) │ B (TTL=30s) │ C (TTL=120s)│                    │
│  │ Expires: 60 │ Expires: 30 │ Expires: 120│                    │
│  └─────────────┴─────────────┴─────────────┘                    │
│                                                                  │
│  Time: 30s → B expires and is removed                           │
│  ┌─────────────┬─────────────┐                                  │
│  │ A (TTL=60s) │ C (TTL=120s)│                                  │
│  │ Expires: 60 │ Expires: 120│                                  │
│  └─────────────┴─────────────┘                                  │
│                                                                  │
│  Time: 60s → A expires and is removed                           │
│  ┌─────────────┐                                                │
│  │ C (TTL=120s)│                                                │
│  │ Expires: 120│                                                │
│  └─────────────┘                                                │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

**Real-world analogy**: 
Food expiration dates. Milk expires after a week regardless of how often you look at it.

| Pros | Cons |
|------|------|
| ✅ Ensures data freshness | ❌ May evict still-useful data |
| ✅ Automatic cleanup | ❌ Requires choosing right TTL |
| ✅ Good for time-sensitive data | ❌ Doesn't consider access patterns |

**Best for**: Session data, API responses, time-sensitive information

---

#### 5. Random Replacement

**Concept**: Remove a random item when cache is full.

**Assumption**: When you can't predict access patterns, random is as good as anything.

| Pros | Cons |
|------|------|
| ✅ Simplest to implement | ❌ Unpredictable behavior |
| ✅ No overhead for tracking | ❌ May evict important items |
| ✅ Works surprisingly well sometimes | ❌ No optimization for patterns |

**Best for**: When access patterns are truly random

---

### 📊 Eviction Policy Comparison

| Policy | Tracks | Complexity | Hit Ratio | Best For |
|--------|--------|------------|-----------|----------|
| LRU | Recency | O(1)* | High | General purpose |
| LFU | Frequency | O(log n) | High (stable) | Stable patterns |
| FIFO | Order | O(1) | Low | Simple cases |
| TTL | Time | O(1) | Medium | Time-sensitive |
| Random | Nothing | O(1) | Low | Unknown patterns |

*With proper data structure (HashMap + Doubly Linked List)

---

## 6. Caching Technologies

### 🔴 Redis (Remote Dictionary Server)

**What**: In-memory data structure store, used as database, cache, and message broker.

**Key Features**:
| Feature | Description |
|---------|-------------|
| Data Structures | Strings, Lists, Sets, Sorted Sets, Hashes, Streams |
| Persistence | RDB snapshots, AOF logging |
| Replication | Master-slave replication |
| Clustering | Redis Cluster for horizontal scaling |
| Pub/Sub | Built-in messaging |
| Lua Scripting | Server-side scripting |
| Transactions | MULTI/EXEC commands |

**When to use Redis**:
- ✅ Need rich data structures (lists, sets, sorted sets)
- ✅ Need persistence options
- ✅ Need pub/sub messaging
- ✅ Need atomic operations
- ✅ Session storage
- ✅ Leaderboards, rate limiting

**Real-world analogy**: 
A Swiss Army knife for caching. It does many things well - not just simple key-value storage.

---

### 🟢 Memcached

**What**: Simple, high-performance, distributed memory caching system.

**Key Features**:
| Feature | Description |
|---------|-------------|
| Data Structure | Simple key-value only |
| Persistence | None (pure cache) |
| Multi-threading | Yes (better CPU utilization) |
| Memory Efficiency | Slab allocation |
| Simplicity | Very simple API |

**When to use Memcached**:
- ✅ Simple key-value caching
- ✅ Need multi-threaded performance
- ✅ Large objects (up to 1MB default)
- ✅ Simple horizontal scaling
- ✅ Don't need persistence

**Real-world analogy**: 
A simple, fast notepad. It does one thing (key-value storage) extremely well.

---

### 📊 Redis vs Memcached

| Aspect | Redis | Memcached |
|--------|-------|-----------|
| Data Types | Rich (strings, lists, sets, etc.) | Simple (strings only) |
| Persistence | Yes (RDB, AOF) | No |
| Replication | Yes | No (client-side) |
| Clustering | Yes (Redis Cluster) | Client-side |
| Threading | Single-threaded* | Multi-threaded |
| Memory | Less efficient | More efficient |
| Max Value Size | 512MB | 1MB (default) |
| Pub/Sub | Yes | No |
| Lua Scripting | Yes | No |
| Use Case | Feature-rich caching | Simple, fast caching |

*Redis 6.0+ has I/O threading

**Decision Guide**:
- Choose **Redis** if you need: data structures, persistence, pub/sub, or complex operations
- Choose **Memcached** if you need: simple caching, multi-threaded performance, or memory efficiency

---

## 7. Distributed Caching

### 🌐 Why Distributed Caching?

Single-server cache limitations:
- ❌ Limited by single server's memory
- ❌ Single point of failure
- ❌ Can't scale horizontally
- ❌ Cache lost on server restart

Distributed cache solves these:
- ✅ Scales across multiple servers
- ✅ High availability
- ✅ Fault tolerance
- ✅ Larger total cache size

### 🔄 Data Distribution Strategies

#### 1. Consistent Hashing

**Problem with simple hashing**:
When you add/remove servers, almost ALL keys need to be remapped.

**Solution**: Consistent hashing minimizes key remapping.

```
┌─────────────────────────────────────────────────────────────────┐
│                    Consistent Hashing Ring                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│                         0°                                       │
│                         │                                        │
│                    ┌────┴────┐                                  │
│                    │ Server A │                                  │
│                    └─────────┘                                  │
│           ╱                         ╲                           │
│         ╱                             ╲                         │
│  270° ─┤                               ├─ 90°                   │
│        │                               │                         │
│   ┌────┴────┐                    ┌────┴────┐                    │
│   │ Server D │                    │ Server B │                    │
│   └─────────┘                    └─────────┘                    │
│         ╲                             ╱                         │
│           ╲                         ╱                           │
│                    ┌─────────┐                                  │
│                    │ Server C │                                  │
│                    └────┬────┘                                  │
│                         │                                        │
│                        180°                                      │
│                                                                  │
│  Key "user:123" hashes to 45° → Goes to Server B                │
│  Key "user:456" hashes to 200° → Goes to Server C               │
│                                                                  │
│  If Server B fails:                                              │
│  • Only keys between A and B need remapping                     │
│  • Other keys stay on their servers                             │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

**Benefits**:
- When a server is added/removed, only K/N keys need remapping (K=keys, N=servers)
- With simple hashing, almost all keys would need remapping

---

#### 2. Replication

**What**: Copying data to multiple nodes for redundancy.

```
┌─────────────────────────────────────────────────────────────────┐
│                    Cache Replication                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Master-Slave Replication:                                       │
│                                                                  │
│       ┌──────────┐                                              │
│       │  Master  │ ◄── All writes go here                       │
│       │  (Redis) │                                              │
│       └────┬─────┘                                              │
│            │                                                     │
│     ┌──────┼──────┐                                             │
│     │      │      │                                              │
│     ▼      ▼      ▼                                              │
│  ┌──────┐ ┌──────┐ ┌──────┐                                     │
│  │Slave1│ │Slave2│ │Slave3│ ◄── Reads distributed here          │
│  └──────┘ └──────┘ └──────┘                                     │
│                                                                  │
│  Benefits:                                                       │
│  • Read scalability (distribute reads across slaves)            │
│  • High availability (promote slave if master fails)            │
│  • Geographic distribution                                       │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

#### 3. Sharding (Partitioning)

**What**: Splitting data across multiple nodes based on key.

```
┌─────────────────────────────────────────────────────────────────┐
│                    Cache Sharding                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Keys are distributed across shards:                             │
│                                                                  │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐              │
│  │   Shard 1   │  │   Shard 2   │  │   Shard 3   │              │
│  │  Keys A-H   │  │  Keys I-P   │  │  Keys Q-Z   │              │
│  │             │  │             │  │             │              │
│  │ user:alice  │  │ user:john   │  │ user:zack   │              │
│  │ user:bob    │  │ user:kate   │  │ user:yuki   │              │
│  │ user:carol  │  │ user:mike   │  │ user:tom    │              │
│  └─────────────┘  └─────────────┘  └─────────────┘              │
│                                                                  │
│  Benefits:                                                       │
│  • Horizontal scaling (add more shards)                         │
│  • Larger total cache size                                       │
│  • Parallel operations                                           │
│                                                                  │
│  Challenges:                                                     │
│  • Cross-shard operations are complex                           │
│  • Resharding when adding nodes                                  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 8. Cache Invalidation

### 🎯 The Hardest Problem in Caching

> "There are only two hard things in Computer Science: cache invalidation and naming things."

**Why it's hard**:
- Cache and database can become inconsistent
- Multiple servers may have different cached values
- Race conditions between updates and reads
- Distributed systems make it even harder

### 📖 Invalidation Strategies

#### 1. TTL-Based Expiration

**How it works**: Data automatically expires after a set time.

```
┌─────────────────────────────────────────────────────────────────┐
│                    TTL-Based Invalidation                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Time 0:    SET user:123 {data} TTL=60s                         │
│             ┌─────────────────────────────────────┐             │
│             │ user:123 = {data}  │ Expires: 60s  │             │
│             └─────────────────────────────────────┘             │
│                                                                  │
│  Time 30s:  GET user:123 → Returns {data} ✓                     │
│                                                                  │
│  Time 60s:  GET user:123 → Returns NULL (expired)               │
│             Cache miss → Fetch fresh data from DB               │
│                                                                  │
│  Pros: Simple, automatic                                         │
│  Cons: Data may be stale until TTL expires                      │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

**Choosing TTL values**:
| Data Type | Suggested TTL | Reason |
|-----------|---------------|--------|
| User sessions | 30 min - 24 hours | Security vs convenience |
| Product catalog | 5-15 minutes | Prices may change |
| Static content | 1 hour - 1 day | Rarely changes |
| Real-time data | 1-5 seconds | Needs freshness |
| Configuration | 1-5 minutes | Balance freshness/load |

---

#### 2. Event-Based Invalidation

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
│          │                                                       │
│          ▼                                                       │
│     ┌─────────┐                                                 │
│     │   DB    │                                                 │
│     └────┬────┘                                                 │
│          │                                                       │
│  3. Publish "user:123 updated" event                            │
│          │                                                       │
│          ▼                                                       │
│     ┌─────────────┐                                             │
│     │ Message Bus │                                             │
│     │  (Kafka)    │                                             │
│     └──────┬──────┘                                             │
│            │                                                     │
│  4. Cache service receives event                                 │
│            │                                                     │
│            ▼                                                     │
│     ┌─────────┐                                                 │
│     │  Cache  │  DELETE user:123                                │
│     └─────────┘                                                 │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

**Pros**: Immediate invalidation, no stale data
**Cons**: Complex infrastructure, potential message loss

---

#### 3. Version-Based Invalidation

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
│  Version stored in:                                              │
│  • Database (source of truth)                                    │
│  • Separate cache key                                            │
│  • Application config                                            │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

**Pros**: No race conditions, simple rollback
**Cons**: Old versions waste space until they expire

---

### ⚠️ Cache Invalidation Patterns

#### Delete vs Update

**Delete (Recommended)**:
1. Update database
2. Delete from cache
3. Next read fetches fresh data

**Update (Risky)**:
1. Update database
2. Update cache with new value

**Why delete is better**:
- Simpler (no need to know new value)
- Avoids race conditions
- Cache only stores what's needed

---

## 9. Cache Stampede Prevention

### 🐘 What is Cache Stampede?

**Also known as**: Thundering herd, cache avalanche, dog-pile effect

**Scenario**:
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

### 🛡️ Prevention Strategies

#### 1. Locking (Mutex)

**How it works**: Only one request fetches from DB; others wait.

```
┌─────────────────────────────────────────────────────────────────┐
│                    Locking Strategy                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Request 1: Cache miss → Acquire lock → Fetch from DB           │
│  Request 2: Cache miss → Lock taken → Wait...                   │
│  Request 3: Cache miss → Lock taken → Wait...                   │
│  Request 4: Cache miss → Lock taken → Wait...                   │
│                                                                  │
│  Request 1: Got data → Store in cache → Release lock            │
│                                                                  │
│  Requests 2,3,4: Lock released → Get from cache (HIT!)          │
│                                                                  │
│  Result: Only 1 DB query instead of 4                           │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

| Pros | Cons |
|------|------|
| ✅ Prevents stampede | ❌ Adds latency for waiting requests |
| ✅ Simple concept | ❌ Lock management complexity |
| ✅ Guaranteed single fetch | ❌ Potential deadlocks |

---

#### 2. Request Coalescing

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

**Real-world analogy**: 
Carpooling. Multiple people going to the same destination share one car instead of each driving separately.

---

#### 3. Probabilistic Early Expiration

**How it works**: Randomly refresh cache before it expires.

```
┌─────────────────────────────────────────────────────────────────┐
│              Probabilistic Early Expiration                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  TTL = 60 seconds                                                │
│                                                                  │
│  Traditional: All requests at T=60s see expiration              │
│               → Stampede!                                        │
│                                                                  │
│  Probabilistic:                                                  │
│  • At T=50s: 10% chance to refresh                              │
│  • At T=55s: 30% chance to refresh                              │
│  • At T=58s: 60% chance to refresh                              │
│  • At T=60s: 100% (expired)                                     │
│                                                                  │
│  Result: One random request refreshes early                     │
│          Others continue using cached value                      │
│          No stampede!                                            │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

#### 4. Background Refresh

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
│  ├─────────────────────────────────────────────────────────────│
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
| ✅ Always fresh data | ❌ Background job management |
| ✅ No stampede possible | ❌ May refresh unused data |

---

## 10. Real-World Examples

### 🐦 Twitter Timeline Caching

**Challenge**: Millions of users, each with personalized timeline

**Solution**:
```
┌─────────────────────────────────────────────────────────────────┐
│                 Twitter Timeline Caching                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Fan-out on Write (for users with < 1M followers):              │
│                                                                  │
│  User A posts tweet                                              │
│       │                                                          │
│       ▼                                                          │
│  ┌─────────────────────────────────────────┐                    │
│  │ Push to all followers' timeline caches  │                    │
│  │                                         │                    │
│  │  Follower 1 cache: [new tweet, ...]    │                    │
│  │  Follower 2 cache: [new tweet, ...]    │                    │
│  │  Follower 3 cache: [new tweet, ...]    │                    │
│  └─────────────────────────────────────────┘                    │
│                                                                  │
│  Fan-out on Read (for celebrities with > 1M followers):         │
│                                                                  │
│  User requests timeline                                          │
│       │                                                          │
│       ▼                                                          │
│  ┌─────────────────────────────────────────┐                    │
│  │ Merge:                                  │                    │
│  │ • Pre-computed timeline (regular users) │                    │
│  │ • Celebrity tweets (fetched on demand)  │                    │
│  └─────────────────────────────────────────┘                    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

### 📺 Netflix Video Metadata

**Challenge**: Billions of requests for video information

**Solution**:
```
┌─────────────────────────────────────────────────────────────────┐
│                Netflix Caching Architecture                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Layer 1: Client Cache (device)                                  │
│  • Recently viewed titles                                        │
│  • User preferences                                              │
│                                                                  │
│  Layer 2: CDN (edge servers)                                     │
│  • Video thumbnails                                              │
│  • Static metadata                                               │
│                                                                  │
│  Layer 3: EVCache (Netflix's distributed cache)                  │
│  • Video metadata                                                │
│  • User viewing history                                          │
│  • Recommendations                                               │
│                                                                  │
│  Layer 4: Database                                               │
│  • Source of truth                                               │
│  • Rarely hit directly                                           │
│                                                                  │
│  Result: 99%+ cache hit ratio                                    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

### 🛒 Amazon Product Catalog

**Challenge**: Millions of products, frequent price changes

**Solution**:
```
┌─────────────────────────────────────────────────────────────────┐
│                Amazon Product Caching                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Separate caches for different data types:                       │
│                                                                  │
│  ┌─────────────────┐  TTL: 24 hours                             │
│  │ Product Details │  (title, description, images)              │
│  │     Cache       │  Changes rarely                            │
│  └─────────────────┘                                            │
│                                                                  │
│  ┌─────────────────┐  TTL: 5 minutes                            │
│  │  Price Cache    │  Changes frequently                        │
│  │                 │  Event-based invalidation                  │
│  └─────────────────┘                                            │
│                                                                  │
│  ┌─────────────────┐  TTL: 1 minute                             │
│  │ Inventory Cache │  Real-time accuracy needed                 │
│  │                 │  Write-through strategy                    │
│  └─────────────────┘                                            │
│                                                                  │
│  ┌─────────────────┐  TTL: 30 minutes                           │
│  │  Review Cache   │  Aggregated data                           │
│  │                 │  Background refresh                        │
│  └─────────────────┘                                            │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 11. Best Practices

### ✅ Do's

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

### ❌ Don'ts

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

### 🔑 Cache Key Design

**Good key format**: `{entity}:{identifier}:{optional-field}`

| Example | Description |
|---------|-------------|
| `user:123` | User with ID 123 |
| `user:123:profile` | User 123's profile |
| `user:123:orders` | User 123's orders |
| `product:456:price` | Product 456's price |
| `session:abc123` | Session with ID abc123 |

**Key design principles**:
- Keep keys short (memory efficient)
- Make keys predictable (easy to invalidate)
- Include version if needed (`user:123:v2`)
- Avoid special characters

---

## 12. Common Pitfalls

### ⚠️ 1. Cache Penetration

**Problem**: Requests for non-existent data always hit database.

```
┌─────────────────────────────────────────────────────────────────┐
│                    Cache Penetration                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Attacker requests: user:999999999 (doesn't exist)              │
│                                                                  │
│  Request → Cache MISS → Database → Not found → No cache         │
│  Request → Cache MISS → Database → Not found → No cache         │
│  Request → Cache MISS → Database → Not found → No cache         │
│  ... (every request hits DB)                                     │
│                                                                  │
│  Solutions:                                                      │
│  1. Cache null results (with short TTL)                         │
│  2. Bloom filter (check if key might exist)                     │
│  3. Rate limiting                                                │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

### ⚠️ 2. Cache Breakdown

**Problem**: Hot key expires, causing sudden DB load.

**Solution**: 
- Never expire hot keys (or use very long TTL)
- Use locking for hot key refresh
- Background refresh before expiration

---

### ⚠️ 3. Cache Avalanche

**Problem**: Many keys expire at the same time.

```
┌─────────────────────────────────────────────────────────────────┐
│                    Cache Avalanche                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Scenario: System restart, all caches populated at same time    │
│                                                                  │
│  Time 0:    All keys set with TTL=60s                           │
│  Time 60s:  ALL keys expire simultaneously!                     │
│             → Massive DB load                                    │
│             → System crash                                       │
│                                                                  │
│  Solutions:                                                      │
│  1. Add random jitter to TTL (TTL = 60 + random(0,10))         │
│  2. Stagger cache warming                                        │
│  3. Use different TTLs for different data types                 │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

### ⚠️ 4. Inconsistency Window

**Problem**: Brief period where cache and DB are inconsistent.

```
┌─────────────────────────────────────────────────────────────────┐
│                  Inconsistency Window                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Time 0ms:   DB has value "A", Cache has value "A"              │
│  Time 10ms:  Update DB to "B"                                   │
│  Time 11ms:  ← INCONSISTENCY WINDOW STARTS                      │
│              DB="B", Cache="A"                                   │
│  Time 15ms:  Delete from cache                                   │
│  Time 16ms:  ← INCONSISTENCY WINDOW ENDS                        │
│                                                                  │
│  During 11-15ms, reads return stale data "A"                    │
│                                                                  │
│  Mitigation:                                                     │
│  • Accept eventual consistency                                   │
│  • Use write-through for strong consistency                     │
│  • Use versioning                                                │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

### ⚠️ 5. Memory Pressure

**Problem**: Cache grows unbounded, causes OOM.

**Solutions**:
- Set max memory limits
- Use appropriate eviction policy
- Monitor memory usage
- Size cache appropriately for workload

---

## 📚 Summary

### Key Takeaways

1. **Caching is essential** for building scalable systems
2. **Choose the right strategy** based on your read/write patterns
3. **Cache invalidation is hard** - plan for it carefully
4. **Monitor everything** - hit ratio, latency, memory
5. **Plan for failure** - system should work without cache
6. **Start simple** - Cache-Aside with TTL works for most cases

### Quick Reference

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

*Happy Caching! 🚀*
