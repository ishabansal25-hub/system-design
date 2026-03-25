# 1. What is Caching?

[← Back to Caching Index](./README.md)

---

## 🎯 Simple Explanation (For Beginners)

Imagine you're a librarian. Every time someone asks for a popular book, you have to walk to the back of the library, find the book, and bring it to them. This takes time!

**Smart solution**: Keep the most popular books on your desk. When someone asks for them, you can hand them over immediately without walking anywhere.

**That's caching!** 📚

In computing terms:
- **Cache** = Your desk (fast, limited space)
- **Database/Disk** = Back of the library (slow, lots of space)
- **Popular books** = Frequently accessed data

---

## 📖 Technical Definition

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

---

## 🔑 Key Terminology

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

---

## 🧠 The Principle of Locality

Caching works because of two fundamental principles:

### 1. Temporal Locality
**"If you accessed something recently, you'll likely access it again soon."**

**Real-world example:**
- You just looked up a word in the dictionary
- You'll probably look at that same page again in the next few minutes
- Keep that page bookmarked!

**System example:**
- User views their profile → likely to view it again
- Cache the profile for quick access

### 2. Spatial Locality
**"If you accessed something, you'll likely access nearby things."**

**Real-world example:**
- You're reading page 50 of a book
- You'll probably read pages 51, 52, 53 next
- Pre-load nearby pages!

**System example:**
- User views product #100 → might view related products
- Cache related products together

---

## 📊 Cache Hit Ratio Formula

```
Hit Ratio = (Cache Hits) / (Cache Hits + Cache Misses) × 100%
```

**Example:**
- 950 cache hits
- 50 cache misses
- Hit Ratio = 950 / (950 + 50) × 100% = **95%**

**Target hit ratios:**
| Application Type | Target Hit Ratio |
|------------------|------------------|
| Web applications | > 90% |
| API responses | > 85% |
| Database queries | > 80% |
| Session data | > 95% |

---

[Next: Why Caching Matters →](./02-why-caching-matters.md)