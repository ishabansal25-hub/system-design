# Redis vs Memcached

[← Back to Caching](./README.md) | [← Previous: Eviction Policies](./05-eviction-policies.md)

---

The two big names in distributed caching. Here's when to use which.

## The short version

**Use Redis** unless you have a specific reason to use Memcached.

Redis does everything Memcached does, plus a lot more. The only reasons to pick Memcached are:
1. You need multi-threaded performance (Redis is mostly single-threaded)
2. You want maximum simplicity
3. You're already using it and it works fine

---

## Redis

Redis started as a cache but evolved into a "data structure server." It's not just key-value - it supports lists, sets, sorted sets, hashes, and more.

**What makes it useful:**

```
# Simple key-value
SET user:123 "John"
GET user:123

# But also lists
LPUSH notifications:123 "New message"
LRANGE notifications:123 0 10

# Sorted sets (great for leaderboards)
ZADD leaderboard 100 "player1"
ZADD leaderboard 200 "player2"
ZREVRANGE leaderboard 0 9  # Top 10

# Hashes (like a mini-document)
HSET user:123 name "John" email "john@example.com"
HGET user:123 name

# Atomic counters
INCR page_views:homepage
```

**Key features:**
- Rich data structures (not just strings)
- Persistence options (RDB snapshots, AOF logging)
- Built-in replication
- Pub/sub messaging
- Lua scripting for complex operations
- Transactions (MULTI/EXEC)
- Clustering for horizontal scaling

**Common use cases:**
- Session storage
- Caching (obviously)
- Rate limiting
- Leaderboards
- Real-time analytics
- Message queues (though Kafka/RabbitMQ are better for serious queuing)
- Pub/sub for real-time features

**The catch:** Single-threaded for command execution. One slow command blocks everything. Redis 6.0+ added I/O threading, but core execution is still single-threaded.

---

## Memcached

Memcached is simpler. It's a pure cache - key-value storage, nothing else.

**What it does:**

```
set user:123 0 3600 4
John

get user:123
```

That's basically it. Set a key with a value and TTL, get it back.

**Key features:**
- Simple key-value only
- Multi-threaded (better CPU utilization)
- Memory-efficient slab allocation
- No persistence (it's a cache, not a database)
- Very simple API

**Common use cases:**
- Simple caching where you don't need Redis's features
- When you want multi-threaded performance
- Large-scale simple caching (Facebook uses it heavily)

**The catch:** No persistence, no data structures, no pub/sub. If you need any of that, you need Redis.

---

## Side-by-side comparison

| | Redis | Memcached |
|---|---|---|
| Data types | Strings, lists, sets, sorted sets, hashes, streams | Strings only |
| Persistence | Yes (RDB, AOF) | No |
| Replication | Built-in | Client-side only |
| Clustering | Redis Cluster | Client-side |
| Threading | Single-threaded* | Multi-threaded |
| Max value size | 512 MB | 1 MB default |
| Pub/Sub | Yes | No |
| Scripting | Lua | No |
| Memory overhead | Higher | Lower |

*Redis 6.0+ has I/O threading

---

## How to choose

**Pick Redis if:**
- You need data structures (lists, sets, sorted sets)
- You want persistence as a safety net
- You need pub/sub
- You need atomic operations beyond simple get/set
- You're not sure (it's the safer default)

**Pick Memcached if:**
- You only need simple key-value caching
- You want multi-threaded performance
- Memory efficiency is critical
- You want maximum simplicity

---

## My take

I've used both in production. Redis wins for most use cases because:

1. **Flexibility.** You start with simple caching, then realize you need a sorted set for a leaderboard. With Redis, you just use it. With Memcached, you need a different solution.

2. **Persistence.** Even for caching, having RDB snapshots means faster recovery after restarts. Your cache warms up from disk instead of from scratch.

3. **Ecosystem.** Better tooling, more documentation, larger community.

The only time I'd pick Memcached is if I had a very specific, simple caching need and wanted to squeeze out maximum performance with multi-threading. But honestly, a Redis cluster usually handles that fine too.

---

## Quick setup examples

**Redis (Docker):**
```bash
docker run -d -p 6379:6379 redis

# Connect
redis-cli
> SET hello world
> GET hello
```

**Memcached (Docker):**
```bash
docker run -d -p 11211:11211 memcached

# Connect (telnet or nc)
telnet localhost 11211
> set hello 0 3600 5
> world
> get hello
```

---

[Next: Distributed Caching →](./07-distributed-caching.md)