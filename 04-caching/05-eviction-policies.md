# Eviction Policies

[← Back to Caching](./README.md) | [← Previous: Caching Strategies](./04-caching-strategies.md)

---

Cache has limited memory. When it's full and you need to add something new, something old has to go. The eviction policy decides what gets kicked out.

---

## LRU (Least Recently Used)

The most common choice. Evict whatever hasn't been accessed for the longest time.

**The logic:** If you haven't used it recently, you probably won't need it soon.

```
Cache: [A, B, C] (capacity 3, C is most recent)

Access D:
→ Cache full, need to evict
→ A is least recently used
→ Evict A
→ Cache: [B, C, D]

Access B:
→ B moves to "most recent" position
→ Cache: [C, D, B]

Access E:
→ C is now least recent
→ Evict C
→ Cache: [D, B, E]
```

**Why it works:** Most access patterns have temporal locality - recently accessed data tends to be accessed again.

**The catch:** A one-time scan of old data can evict your actually-useful cached items. If you iterate through a million records once, LRU will evict everything else to make room for data you'll never access again.

**Implementation:** HashMap + Doubly Linked List gives you O(1) for both get and put. This is a classic interview question.

**When to use:** Default choice for most applications.

---

## LFU (Least Frequently Used)

Evict whatever has been accessed the fewest times.

**The logic:** Frequently accessed data is more valuable than rarely accessed data.

```
Cache with access counts:
A: 100 accesses
B: 5 accesses  
C: 3 accesses

Need to evict → C goes (lowest count)
```

**Why it works:** Keeps your genuinely popular data around.

**The catch:** 
- New items start with low counts and might get evicted before they have a chance to prove they're useful
- Old items that were popular months ago but aren't anymore will stick around forever

Some implementations decay counts over time to fix the second problem.

**When to use:** Stable workloads where access patterns don't change much. Not great for trending/viral content.

---

## FIFO (First In, First Out)

Evict the oldest item, regardless of how often it's accessed.

```
Cache: [A, B, C] (A was added first)

Add D → Evict A → [B, C, D]
Add E → Evict B → [C, D, E]
```

**Why you might use it:** Dead simple to implement. Just a queue.

**The catch:** Completely ignores access patterns. Your most popular item gets evicted just because it was added first.

**When to use:** Honestly, rarely. Maybe if all your data has similar access patterns and you want simplicity.

---

## TTL (Time To Live)

Not really an eviction policy, but often used alongside one. Items expire after a set time.

```
SET user:123 {data} TTL=300  # Expires in 5 minutes

After 5 minutes → automatically removed
```

**Choosing TTL values:**

| Data type | TTL | Why |
|-----------|-----|-----|
| User sessions | 30 min - 24 hours | Security vs convenience tradeoff |
| Product prices | 5-15 minutes | Prices change, but not every second |
| User profiles | 1-5 minutes | Changes are infrequent |
| Static config | 1 hour+ | Rarely changes |
| Real-time data | Seconds | Needs to be fresh |

**Pro tip:** Add some randomness to TTLs to prevent stampedes. Instead of all keys expiring at exactly 5 minutes, use 4-6 minutes randomly.

---

## Random

Just pick something random to evict.

**Why it exists:** Zero overhead. No tracking needed.

**When to use:** When you genuinely have no idea about access patterns, or when the overhead of tracking isn't worth it. Surprisingly, random eviction performs better than you'd expect in some workloads.

---

## Which one to use?

**Short answer:** LRU with TTL. It's the default for a reason.

```
Redis default: LRU (with several variants)
Memcached default: LRU
Most cache libraries: LRU
```

**Longer answer:**

| Situation | Policy |
|-----------|--------|
| General purpose | LRU |
| Stable, predictable access | LFU |
| Time-sensitive data | TTL (with LRU as backup) |
| No idea / don't care | Random or FIFO |

In practice, LRU + TTL covers 95% of use cases. You set a TTL so stale data eventually expires, and LRU handles eviction when the cache is full.

---

## LRU implementation (interview favorite)

If you're asked to implement LRU cache in an interview, here's the approach:

```
Data structures:
1. HashMap: key → node (for O(1) lookup)
2. Doubly Linked List: maintains order (head = least recent, tail = most recent)

Get(key):
- Look up in HashMap
- If found, move node to tail (most recent)
- Return value

Put(key, value):
- If key exists, update value, move to tail
- If key doesn't exist:
  - If at capacity, remove head (least recent), delete from HashMap
  - Add new node at tail
  - Add to HashMap
```

Both operations are O(1). The doubly linked list lets you remove any node in O(1) if you have a reference to it (which the HashMap provides).

---

[Next: Caching Technologies →](./06-caching-technologies.md)