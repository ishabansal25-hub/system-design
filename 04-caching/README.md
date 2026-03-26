# Caching

> "There are only two hard things in Computer Science: cache invalidation and naming things." — Phil Karlton

I went deep on caching because it's one of those things that seems simple until you actually have to implement it in production. Then you realize there's a lot of nuance.

---

## If you're prepping for interviews

I made a condensed cheatsheet: **[Interview Cheatsheet](./INTERVIEW-CHEATSHEET.md)**

It's meant for a quick 1-2 day review. Has all the key concepts without the deep dives.

---

## What's covered

| # | Topic | What it covers |
|---|-------|----------------|
| 1 | [What is Caching?](./01-what-is-caching.md) | The basics - terminology, locality principles |
| 2 | [Why Caching Matters](./02-why-caching-matters.md) | Performance gains, cost savings, when NOT to cache |
| 3 | [Types of Caches](./03-types-of-caches.md) | Browser, CDN, app-level, database caches |
| 4 | [Caching Strategies](./04-caching-strategies.md) | Cache-Aside, Read-Through, Write-Through, etc. |
| 5 | [Eviction Policies](./05-eviction-policies.md) | LRU, LFU, FIFO, TTL |
| 6 | [Redis vs Memcached](./06-caching-technologies.md) | When to use which |
| 7 | [Distributed Caching](./07-distributed-caching.md) | Consistent hashing, replication, sharding |
| 8 | [Cache Invalidation](./08-cache-invalidation.md) | The hard part |
| 9 | [Cache Stampede](./09-cache-stampede.md) | Thundering herd and how to prevent it |
| 10 | [Real-World Examples](./10-real-world-examples.md) | How Twitter, Netflix, Amazon do it |
| 11 | [Best Practices](./11-best-practices.md) | Lessons learned, common mistakes |
| 12 | [Scenarios](./12-real-world-scenarios.md) | 7 detailed problems with solutions |

---

## Where to start

**If you're new to caching:**
Start with [What is Caching?](./01-what-is-caching.md) and work through the first three files. They build on each other.

**If you're building something:**
Jump to [Caching Strategies](./04-caching-strategies.md) to pick the right approach, then [Redis vs Memcached](./06-caching-technologies.md) to choose your tech.

**If you have an interview coming up:**
The [Scenarios](./12-real-world-scenarios.md) file has the kind of questions that actually come up. Things like "your hot cache key expires and 50k requests hit the DB" - not just textbook definitions.

---

## Quick reference

For most cases, this works:

| Situation | What I'd do |
|-----------|-------------|
| General purpose | Cache-Aside + LRU + TTL |
| Read-heavy API | Read-Through + LRU |
| Write-heavy | Write-Behind with batching |
| Need strong consistency | Write-Through |
| User sessions | Redis with TTL |
| Static assets | CDN |

---

## My take on caching

After writing all this, here's what I think matters most:

1. **Start simple.** Cache-Aside with TTL handles 80% of use cases. Don't over-engineer.

2. **Invalidation is where things break.** Spend more time thinking about when to invalidate than how to cache.

3. **Monitor your hit ratio.** If it's below 80%, something's wrong - either you're caching the wrong things or your TTLs are off.

4. **Plan for cache failure.** Your system should work (slowly) without the cache. If Redis goes down, users should still be able to use your app.

5. **Cache isn't free.** Memory costs money. Don't cache everything just because you can.

---

[Back to main notes](../README.md)