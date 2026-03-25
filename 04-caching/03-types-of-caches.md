# 3. Types of Caches

[← Back to Caching Index](./README.md) | [← Previous: Why Caching Matters](./02-why-caching-matters.md)

---

## 🏗️ Cache Hierarchy

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

---

## 📱 1. Client-Side Cache

**What**: Data stored on user's device (browser, mobile app)

**Examples**:
- Browser cache (images, CSS, JS files)
- LocalStorage / SessionStorage
- Mobile app SQLite cache
- Service Workers (PWA offline cache)

### Characteristics

| Aspect | Details |
|--------|---------|
| Location | User's device |
| Control | Limited (user can clear) |
| Size | ~5-50 MB typical |
| Speed | Fastest (no network) |
| Scope | Single user only |

**Real-world analogy**: Your phone's photo gallery. Photos are stored locally so you can view them instantly without downloading again.

---

## 🌐 2. CDN Cache (Content Delivery Network)

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

### What CDNs cache
- Static files (images, videos, CSS, JS)
- API responses (with proper headers)
- HTML pages (for static sites)

### Popular CDN providers

| Provider | Strengths |
|----------|-----------|
| CloudFront | AWS integration, Lambda@Edge |
| Cloudflare | DDoS protection, Workers |
| Akamai | Enterprise, largest network |
| Fastly | Real-time purging, edge compute |

**Real-world analogy**: Amazon warehouses. Instead of shipping everything from one central warehouse, they have fulfillment centers near major cities for faster delivery.

---

## 🖥️ 3. Application-Level Cache

**What**: Cache within your application servers

### a) In-Memory Cache (Single Server)

**What**: Data stored in application's RAM

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

**Real-world analogy**: Your brain's short-term memory. Quick to access but limited capacity and forgotten when you sleep.

### b) Distributed Cache (Multiple Servers)

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

| Aspect | Details |
|--------|---------|
| Speed | Fast (sub-millisecond) |
| Scope | All servers in cluster |
| Persistence | Configurable |
| Size | Scales with nodes |

**Real-world analogy**: A shared Google Doc. Everyone on the team can access and update it, and changes are visible to all.

---

## 🗄️ 4. Database Cache

**What**: Caching at the database level

| Type | Description | Example |
|------|-------------|---------|
| Query Cache | Caches query results | Same SELECT returns cached result |
| Buffer Pool | Caches data pages in memory | Frequently accessed rows stay in RAM |
| Connection Pool | Reuses database connections | Avoids connection overhead |
| Materialized Views | Pre-computed query results | Complex aggregations stored as tables |

**Real-world analogy**: A restaurant's prep station. Commonly used ingredients are pre-chopped and ready, rather than preparing everything from scratch for each order.

---

## 📊 Cache Type Comparison

| Type | Speed | Scope | Size | Control | Best For |
|------|-------|-------|------|---------|----------|
| Client-side | Fastest | Single user | Small | Limited | Static assets, user prefs |
| CDN | Very fast | Global | Large | Medium | Static content, media |
| In-memory | Fast | Single server | Medium | Full | Hot data, sessions |
| Distributed | Fast | All servers | Large | Full | Shared state, sessions |
| Database | Medium | Database | Large | Limited | Query results |

---

[Next: Caching Strategies →](./04-caching-strategies.md)