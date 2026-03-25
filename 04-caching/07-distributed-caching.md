# 7. Distributed Caching

[← Back to Caching Index](./README.md) | [← Previous: Caching Technologies](./06-caching-technologies.md)

---

## 🌐 Why Distributed Caching?

**Single-server cache limitations:**
- ❌ Limited by single server's memory
- ❌ Single point of failure
- ❌ Can't scale horizontally
- ❌ Cache lost on server restart

**Distributed cache solves these:**
- ✅ Scales across multiple servers
- ✅ High availability
- ✅ Fault tolerance
- ✅ Larger total cache size

---

## 🔄 Data Distribution Strategies

### 1. Consistent Hashing

**Problem with simple hashing**: When you add/remove servers, almost ALL keys need to be remapped.

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

**Benefits**: When a server is added/removed, only K/N keys need remapping (K=keys, N=servers)

---

### 2. Replication

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

### 3. Sharding (Partitioning)

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

[Next: Cache Invalidation →](./08-cache-invalidation.md)