# What is Caching?

[← Back to Caching](./README.md)

---

## The simple version

You know how you keep your most-used apps on your phone's home screen instead of digging through folders every time? That's basically caching.

In software terms: instead of fetching data from a slow source (database, API, disk) every single time, you keep a copy somewhere fast (memory) so subsequent requests are instant.

```
Without cache:
User → App → Database (slow, every time)

With cache:
User → App → Cache (fast!) → only goes to DB on cache miss
```

The first request still hits the database. But the second, third, hundredth request? Served from memory in microseconds instead of milliseconds.

---

## The terminology

These come up constantly, so worth memorizing:

| Term | What it means |
|------|---------------|
| **Cache Hit** | Found it in the cache. Fast path. |
| **Cache Miss** | Not in cache, had to fetch from source. Slow path. |
| **Hit Ratio** | What % of requests are hits. Higher = better. |
| **TTL** | Time To Live. How long before cached data expires. |
| **Eviction** | Kicking something out of cache to make room. |
| **Warm Cache** | Cache that's been running and has data in it. |
| **Cold Cache** | Empty cache, like right after a restart. |
| **Stale Data** | Cached data that's out of date with the source. |

**Hit ratio** is the one you'll care about most in production. If you're at 95%, that means 95 out of 100 requests never touch your database. That's huge.

```
Hit Ratio = Cache Hits / (Cache Hits + Cache Misses)
```

What's a good hit ratio? Depends on your use case, but roughly:
- 90%+ for web apps
- 85%+ for APIs  
- 95%+ for session data

If you're below 80%, something's probably wrong with your caching strategy.

---

## Why caching works (locality)

Caching isn't magic - it works because of how people actually use software. Two patterns:

### Temporal locality
"If you accessed something recently, you'll probably access it again soon."

Think about it: when you check your email, you probably check it again in the next hour. When you view a product page, you might view it again while comparing options. When you load your Twitter feed, you'll probably refresh it.

Recent data tends to be accessed again. So we cache it.

### Spatial locality  
"If you accessed something, you'll probably access related things."

If you're reading page 50 of a book, you'll probably read page 51 next. If you're looking at a user's profile, you might look at their posts. If you're viewing a product, you might view similar products.

This is why some systems pre-fetch related data into cache.

---

## A quick example

Let's say you have a user profile endpoint that gets hit 1000 times per second.

**Without caching:**
- 1000 database queries per second
- Each query takes 50ms
- Database is constantly under load
- Users wait 50ms+ for every request

**With caching (95% hit ratio):**
- 50 database queries per second (only the misses)
- 950 requests served from cache in ~1ms
- Database load reduced by 95%
- Most users get responses in 1-2ms

That's the difference caching makes. It's not about making individual queries faster - it's about not making the query at all.

---

## What caching is NOT

A few misconceptions I've seen:

**It's not a database replacement.** Cache is temporary. It can be evicted, it can fail, it can restart empty. Your source of truth is still the database.

**It's not always the right answer.** If your data changes constantly, or if every request is unique, caching might not help much. More on this in the "why caching matters" section.

**It's not free.** Memory costs money. Cache invalidation is complex. There's operational overhead. It's a tradeoff, not a magic solution.

---

## The cache flow

Here's what typically happens with a cache-aside pattern (the most common):

```
1. Request comes in for user #123
2. Check cache: "Do we have user #123?"
3a. YES (cache hit) → Return cached data. Done.
3b. NO (cache miss) → Continue to step 4
4. Query database for user #123
5. Store result in cache (with TTL)
6. Return data to user
```

Next time someone requests user #123, step 3a kicks in and we skip the database entirely.

The tricky parts:
- What TTL do you set? Too short = low hit ratio. Too long = stale data.
- What happens when user #123 updates their profile? Cache still has old data.
- What if cache is full? Something has to be evicted.

These are the problems the rest of this guide addresses.

---

[Next: Why Caching Matters →](./02-why-caching-matters.md)