# Types of Caches

[← Back to Caching](./README.md) | [← Previous: Why Caching Matters](./02-why-caching-matters.md)

---

Caching happens at multiple layers. Understanding where each type fits helps you decide what to use.

```
User's Device
    ↓
CDN (edge servers)
    ↓
Your App Servers (in-memory or distributed)
    ↓
Database (query cache, buffer pool)
```

Each layer catches requests before they hit the next one. The closer to the user, the faster.

---

## 1. Client-Side Cache

This is caching on the user's device - browser, mobile app, desktop app.

**What gets cached:**
- Images, CSS, JavaScript (browser cache)
- LocalStorage / SessionStorage data
- Mobile app's local database
- Service Worker cache (for offline PWAs)

**The good:**
- Fastest possible - no network at all
- Reduces your server load
- Works offline

**The bad:**
- You can't control it (users can clear it)
- Limited space (~5-50 MB typically)
- Only helps that one user

**When to use:** Static assets, user preferences, anything that doesn't change often and is specific to that user.

---

## 2. CDN Cache

CDN = Content Delivery Network. Servers distributed around the world that cache your content close to users.

**How it works:**

```
User in Tokyo requests image.jpg
    ↓
CDN edge server in Tokyo: "Do I have it?"
    ↓
Yes → Return immediately (20ms)
No → Fetch from your origin server in US (200ms), cache it, return
    ↓
Next user in Tokyo gets it in 20ms
```

**What CDNs cache:**
- Static files (images, videos, CSS, JS)
- API responses (if you set the right headers)
- Entire HTML pages (for static sites)

**Popular options:**
- CloudFront (AWS) - good if you're already on AWS
- Cloudflare - great free tier, good DDoS protection
- Fastly - real-time purging, used by GitHub/Stripe
- Akamai - enterprise, massive network

**When to use:** Any static content, especially if you have users globally. The latency improvement from serving from a nearby edge server is significant.

---

## 3. Application-Level Cache

This is caching in your application code. Two flavors:

### In-Memory (Local) Cache

Data stored in your app server's RAM. Think a Python dictionary or Java HashMap.

```python
# Simple example
cache = {}

def get_user(user_id):
    if user_id in cache:
        return cache[user_id]
    user = db.fetch_user(user_id)
    cache[user_id] = user
    return user
```

**Characteristics:**
- Extremely fast (nanoseconds)
- Only accessible by that one server
- Lost when server restarts
- Limited by server RAM

**When to use:** 
- Single-server apps
- Data that's OK to be different across servers briefly
- Extremely hot data where even 1ms Redis latency matters

**Libraries:** Guava Cache (Java), Caffeine (Java), cachetools (Python)

### Distributed Cache

Shared cache that all your app servers can access. Usually Redis or Memcached.

```
App Server 1 ─┐
App Server 2 ─┼──→ Redis Cluster ──→ Database
App Server 3 ─┘
```

**Characteristics:**
- Fast (sub-millisecond, but slower than local)
- Shared across all servers
- Survives individual server restarts
- Scales horizontally

**When to use:**
- Multi-server deployments
- Session storage
- Any shared state
- Most production caching needs

---

## 4. Database Cache

Your database does its own caching too.

**Query Cache:** Some databases cache query results. Same query = cached result. (MySQL had this, but it's deprecated because invalidation was problematic.)

**Buffer Pool:** The database keeps frequently accessed data pages in RAM. This is why giving your database more RAM often helps performance.

**Connection Pool:** Not exactly a cache, but reusing database connections avoids the overhead of establishing new ones.

**Materialized Views:** Pre-computed query results stored as tables. Good for expensive aggregations that don't need real-time data.

You usually don't configure these directly (except buffer pool size), but it's good to know they exist.

---

## Comparison

| Type | Speed | Scope | Best for |
|------|-------|-------|----------|
| Client-side | Fastest | Single user | Static assets, offline |
| CDN | Very fast | Global | Static content, media |
| Local (in-memory) | Fast | Single server | Hot data, single-server apps |
| Distributed | Fast | All servers | Shared state, sessions |
| Database | Medium | Database | Automatic, query results |

---

## What I'd use

**Small app, single server:** Local in-memory cache is fine. Keep it simple.

**Multi-server app:** Redis for most things. It's the default choice for a reason.

**Static assets:** CDN. Always. The cost is minimal and the benefit is huge.

**Global users:** CDN + Redis. CDN for static stuff, Redis for dynamic data.

**Extremely hot data:** Two-tier: local cache in front of Redis. But only if you've measured and the 1ms Redis latency actually matters.

Don't over-engineer. Start with one layer (probably Redis), measure, then add more if needed.

---

[Next: Caching Strategies →](./04-caching-strategies.md)