# 5. Cache Eviction Policies

[← Back to Caching Index](./README.md) | [← Previous: Caching Strategies](./04-caching-strategies.md)

---

## 🎯 Why Eviction is Needed

Cache has limited memory. When it's full, we must remove something to add new data. The eviction policy decides **what to remove**.

```
┌─────────────────────────────────────────────────────────────────┐
│                    Cache Eviction Scenario                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Cache is FULL (capacity: 5 items)                              │
│  ┌─────┬─────┬─────┬─────┬─────┐                               │
│  │  A  │  B  │  C  │  D  │  E  │                               │
│  └─────┴─────┴─────┴─────┴─────┘                               │
│                                                                  │
│  New item F arrives...                                           │
│                                                                  │
│  Question: Which item should be removed?                         │
│  Answer depends on the EVICTION POLICY                          │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 📖 1. LRU (Least Recently Used)

**Concept**: Remove the item that hasn't been accessed for the longest time.

**Assumption**: If you haven't used it recently, you probably won't need it soon.

```
┌─────────────────────────────────────────────────────────────────┐
│                         LRU Example                              │
├─────────────────────────────────────────────────────────────────┤
│  Cache capacity: 3                                               │
│                                                                  │
│  Access sequence: A, B, C, A, D                                  │
│                                                                  │
│  Step 1: Access A    [A]           (A is most recent)           │
│  Step 2: Access B    [A, B]        (B is most recent)           │
│  Step 3: Access C    [A, B, C]     (C is most recent) - FULL    │
│  Step 4: Access A    [B, C, A]     (A moves to most recent)     │
│  Step 5: Access D    [C, A, D]     (B evicted - least recent)   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

**Real-world analogy**: Your browser's "recently visited" list.

| Pros | Cons |
|------|------|
| ✅ Simple to understand | ❌ Doesn't consider frequency |
| ✅ Works well for most cases | ❌ One-time access can evict useful data |
| ✅ O(1) with proper implementation | ❌ Scan resistance issues |

**Best for**: General-purpose caching, web applications

---

## 📖 2. LFU (Least Frequently Used)

**Concept**: Remove the item that has been accessed the fewest times.

```
┌─────────────────────────────────────────────────────────────────┐
│                         LFU Example                              │
├─────────────────────────────────────────────────────────────────┤
│  Cache capacity: 3                                               │
│                                                                  │
│  Access sequence: A, A, A, B, B, C, D                            │
│                                                                  │
│  After A, A, A:   [A(3)]                                        │
│  After B, B:      [A(3), B(2)]                                  │
│  After C:         [A(3), B(2), C(1)]    - FULL                  │
│  After D:         [A(3), B(2), D(1)]    - C evicted (freq=1)    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

**Real-world analogy**: A store keeping frequently-sold products on shelves.

| Pros | Cons |
|------|------|
| ✅ Keeps frequently used items | ❌ New items may be evicted quickly |
| ✅ Good for stable access patterns | ❌ Old popular items may never be evicted |

**Best for**: Stable workloads, when access patterns don't change much

---

## 📖 3. FIFO (First In, First Out)

**Concept**: Remove the oldest item (first one added).

```
┌─────────────────────────────────────────────────────────────────┐
│                        FIFO Example                              │
├─────────────────────────────────────────────────────────────────┤
│  Cache capacity: 3                                               │
│                                                                  │
│  Step 1: Add A     [A]                                          │
│  Step 2: Add B     [A, B]                                       │
│  Step 3: Add C     [A, B, C]     - FULL                         │
│  Step 4: Add D     [B, C, D]     - A evicted (first in)         │
│                                                                  │
│  Note: Even if A was accessed 100 times, it still gets evicted  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

| Pros | Cons |
|------|------|
| ✅ Very simple to implement | ❌ Ignores access patterns |
| ✅ Predictable behavior | ❌ May evict frequently used items |

**Best for**: Simple scenarios, when all items have similar access patterns

---

## 📖 4. TTL (Time To Live)

**Concept**: Each item has an expiration time. Remove when expired.

```
┌─────────────────────────────────────────────────────────────────┐
│                         TTL Example                              │
├─────────────────────────────────────────────────────────────────┤
│  Time 0:    SET items with different TTLs                       │
│  ┌─────────────┬─────────────┬─────────────┐                    │
│  │ A (TTL=60s) │ B (TTL=30s) │ C (TTL=120s)│                    │
│  └─────────────┴─────────────┴─────────────┘                    │
│                                                                  │
│  Time 30s → B expires and is removed                            │
│  Time 60s → A expires and is removed                            │
│  Time 120s → C expires and is removed                           │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

**Choosing TTL values**:
| Data Type | Suggested TTL | Reason |
|-----------|---------------|--------|
| User sessions | 30 min - 24 hours | Security vs convenience |
| Product catalog | 5-15 minutes | Prices may change |
| Static content | 1 hour - 1 day | Rarely changes |
| Real-time data | 1-5 seconds | Needs freshness |

---

## 📖 5. Random Replacement

**Concept**: Remove a random item when cache is full.

| Pros | Cons |
|------|------|
| ✅ Simplest to implement | ❌ Unpredictable behavior |
| ✅ No overhead for tracking | ❌ May evict important items |

**Best for**: When access patterns are truly random

---

## 📊 Eviction Policy Comparison

| Policy | Tracks | Complexity | Hit Ratio | Best For |
|--------|--------|------------|-----------|----------|
| LRU | Recency | O(1)* | High | General purpose |
| LFU | Frequency | O(log n) | High (stable) | Stable patterns |
| FIFO | Order | O(1) | Low | Simple cases |
| TTL | Time | O(1) | Medium | Time-sensitive |
| Random | Nothing | O(1) | Low | Unknown patterns |

*With proper data structure (HashMap + Doubly Linked List)

---

[Next: Caching Technologies →](./06-caching-technologies.md)