# 6. Caching Technologies

[← Back to Caching Index](./README.md) | [← Previous: Eviction Policies](./05-eviction-policies.md)

---

## 🔴 Redis (Remote Dictionary Server)

**What**: In-memory data structure store, used as database, cache, and message broker.

### Key Features

| Feature | Description |
|---------|-------------|
| Data Structures | Strings, Lists, Sets, Sorted Sets, Hashes, Streams |
| Persistence | RDB snapshots, AOF logging |
| Replication | Master-slave replication |
| Clustering | Redis Cluster for horizontal scaling |
| Pub/Sub | Built-in messaging |
| Lua Scripting | Server-side scripting |
| Transactions | MULTI/EXEC commands |

### When to use Redis
- ✅ Need rich data structures (lists, sets, sorted sets)
- ✅ Need persistence options
- ✅ Need pub/sub messaging
- ✅ Need atomic operations
- ✅ Session storage
- ✅ Leaderboards, rate limiting

**Real-world analogy**: A Swiss Army knife for caching. It does many things well - not just simple key-value storage.

---

## 🟢 Memcached

**What**: Simple, high-performance, distributed memory caching system.

### Key Features

| Feature | Description |
|---------|-------------|
| Data Structure | Simple key-value only |
| Persistence | None (pure cache) |
| Multi-threading | Yes (better CPU utilization) |
| Memory Efficiency | Slab allocation |
| Simplicity | Very simple API |

### When to use Memcached
- ✅ Simple key-value caching
- ✅ Need multi-threaded performance
- ✅ Large objects (up to 1MB default)
- ✅ Simple horizontal scaling
- ✅ Don't need persistence

**Real-world analogy**: A simple, fast notepad. It does one thing (key-value storage) extremely well.

---

## 📊 Redis vs Memcached

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

### Decision Guide

**Choose Redis if you need:**
- Data structures beyond key-value
- Persistence
- Pub/sub messaging
- Complex operations (sorted sets, etc.)

**Choose Memcached if you need:**
- Simple caching
- Multi-threaded performance
- Memory efficiency
- Simplicity

---

[Next: Distributed Caching →](./07-distributed-caching.md)