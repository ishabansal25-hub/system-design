# 10. Real-World Examples

[← Back to Caching Index](./README.md) | [← Previous: Cache Stampede](./09-cache-stampede.md)

---

## 🐦 Twitter Timeline Caching

**Challenge**: Millions of users, each with personalized timeline

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

## 📺 Netflix Video Metadata

**Challenge**: Billions of requests for video information

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

## 🛒 Amazon Product Catalog

**Challenge**: Millions of products, frequent price changes

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

## 📱 Facebook Social Graph

**Challenge**: Billions of users, complex relationships

**Solution:**
- **TAO (The Associations and Objects)**: Custom distributed cache
- Caches social graph (friends, likes, comments)
- 99.9%+ cache hit ratio
- Handles billions of queries per second

**Key insight**: Different TTLs for different relationship types
- Friend relationships: Long TTL (rarely change)
- Like counts: Short TTL (change frequently)
- Comments: Medium TTL with event invalidation

---

[Next: Best Practices & Pitfalls →](./11-best-practices.md)