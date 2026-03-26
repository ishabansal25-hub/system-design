# Distributed Caching

[← Back to Caching](./README.md) | [← Previous: Redis vs Memcached](./06-caching-technologies.md)

---

A single Redis server can only hold so much data and handle so many requests. When you need more, you distribute the cache across multiple nodes.

## Why go distributed?

**Single server limitations:**
- Memory capped at what one machine can hold
- Single point of failure
- CPU/network bottleneck under high load

**Distributed cache gives you:**
- More total memory (sum of all nodes)
- Fault tolerance (one node dies, others keep working)
- Higher throughput (requests spread across nodes)

---

## Consistent Hashing

This is the key concept for distributing data across nodes. It comes up in interviews constantly.

**The problem with simple hashing:**

Say you have 3 cache servers and use `hash(key) % 3` to pick which server stores each key.

```
hash("user:1") % 3 = 0 → Server 0
hash("user:2") % 3 = 1 → Server 1
hash("user:3") % 3 = 2 → Server 2
```

Now you add a 4th server. Suddenly it's `hash(key) % 4`:

```
hash("user:1") % 4 = 1 → Server 1 (was 0!)
hash("user:2") % 4 = 2 → Server 2 (was 1!)
hash("user:3") % 4 = 3 → Server 3 (was 2!)
```

Almost every key moved to a different server. Your cache is effectively empty - everything needs to be re-fetched from the database. This is a disaster.

**Consistent hashing fixes this:**

Imagine a ring (0 to 360 degrees, or 0 to 2^32). Both servers and keys are hashed onto this ring.

```
        0°
        │
   Server A (at 45°)
       ╱
      ╱
270° ─────────── 90°
      ╲
       ╲
   Server B (at 200°)
        │
       180°
```

To find which server stores a key:
1. Hash the key to get a position on the ring
2. Walk clockwise until you hit a server
3. That server owns the key

```
hash("user:1") = 30° → walk clockwise → hits Server A at 45°
hash("user:2") = 100° → walk clockwise → hits Server B at 200°
```

**Why this is better:**

When you add Server C at 120°:
- Keys between 90° and 120° move from Server B to Server C
- Everything else stays put

Only ~1/N of keys move when adding a node (N = number of nodes). Much better than "everything moves."

**Virtual nodes:**

In practice, each physical server gets multiple positions on the ring (virtual nodes). This gives better distribution and handles the case where servers have different capacities.

---

## Replication

Copying data to multiple nodes for redundancy.

**Master-Replica setup:**

```
Writes → Master
              ↓ (async replication)
         ┌────┼────┐
         ↓    ↓    ↓
      Replica Replica Replica ← Reads distributed here
```

**How it helps:**
- Read scalability: spread reads across replicas
- Fault tolerance: if master dies, promote a replica
- Geographic distribution: replicas in different regions

**The tradeoff:** Replication lag. Writes go to master, then propagate to replicas. For a brief moment, replicas have stale data. Usually milliseconds, but it matters for some use cases.

---

## Sharding

Splitting data across nodes so each node holds a subset.

```
Shard 1: keys A-H
Shard 2: keys I-P  
Shard 3: keys Q-Z
```

Or more commonly, using consistent hashing to determine which shard owns which keys.

**Benefits:**
- Total capacity = sum of all shards
- Parallel operations (different keys hit different shards)
- Each shard handles less load

**Challenges:**
- Cross-shard operations are hard (what if you need data from multiple shards?)
- Adding shards requires resharding (moving data around)
- Hot shards: if one key range is way more popular, that shard gets hammered

---

## Redis Cluster

Redis has built-in clustering that handles sharding and replication.

**How it works:**
- 16,384 hash slots distributed across nodes
- Each key hashes to a slot: `CRC16(key) % 16384`
- Each node owns a range of slots
- Each node can have replicas

```
Node 1: slots 0-5460 (+ replica)
Node 2: slots 5461-10922 (+ replica)
Node 3: slots 10923-16383 (+ replica)
```

**What you get:**
- Automatic sharding
- Automatic failover (replica promoted if master dies)
- Clients are redirected to the right node

**What you don't get:**
- Multi-key operations across different slots (unless you use hash tags)
- Transactions across nodes

---

## Practical considerations

**Start simple.** A single Redis instance handles a lot more than you'd think. Don't distribute until you actually need to.

**Measure first.** Is your bottleneck memory, CPU, or network? Sharding helps with memory and CPU. Replication helps with read throughput.

**Plan for failure.** What happens when a node dies? With replication, you failover. Without it, you lose that portion of your cache and hit the database harder.

**Consider managed services.** AWS ElastiCache, Redis Cloud, etc. handle the operational complexity of clustering. Worth it unless you have specific reasons to self-manage.

---

[Next: Cache Invalidation →](./08-cache-invalidation.md)