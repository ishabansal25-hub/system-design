# How Big Companies Do Caching

[← Back to Caching](./README.md) | [← Previous: Cache Stampede](./09-cache-stampede.md)

---

Looking at how Twitter, Netflix, Amazon, and Facebook handle caching gives you a sense of what works at scale. These aren't just theoretical - they're battle-tested approaches.

---

## Twitter: Timeline Caching

Twitter's main challenge: showing each user a personalized timeline of tweets from people they follow.

**The naive approach would be:**
```
User opens app
→ Query: "Get all tweets from people I follow, sorted by time"
→ This is expensive (joins across millions of rows)
```

**What they actually do:**

For regular users (< ~1M followers), they use **fan-out on write**:

```
User A posts a tweet
    ↓
System looks up all of A's followers
    ↓
Pushes the tweet ID to each follower's timeline cache
    ↓
Follower's timeline cache: [tweet_id_new, tweet_id_2, tweet_id_3, ...]
```

When you open the app, your timeline is already pre-computed. Just fetch the tweet IDs from your cache and hydrate with tweet content.

**The celebrity problem:**

If someone has 50 million followers, you can't push to 50 million caches every time they tweet. So for celebrities, they use **fan-out on read**:

```
User opens app
    ↓
Fetch pre-computed timeline (from regular users they follow)
    ↓
Fetch recent tweets from celebrities they follow (on demand)
    ↓
Merge and display
```

It's a hybrid approach. Most of the work is done at write time, but celebrity tweets are fetched at read time.

---

## Netflix: Multi-Layer Caching

Netflix serves billions of requests for video metadata, thumbnails, and recommendations.

**Their caching layers:**

```
Layer 1: Client (your device)
- Recently viewed titles
- Your profile data
- Cached locally, survives app restarts

Layer 2: CDN (edge servers worldwide)
- Video thumbnails
- Static assets
- Served from server nearest to you

Layer 3: EVCache (their distributed cache)
- Video metadata
- Viewing history
- Recommendations
- This is where most of the magic happens

Layer 4: Database
- Source of truth
- Rarely hit directly for reads
```

**EVCache** is Netflix's custom distributed cache built on Memcached. It's designed for:
- Multi-region replication
- High availability (data replicated across zones)
- Tunable consistency

They achieve 99%+ cache hit ratios. The database is mostly for writes and the occasional cache miss.

**Key insight:** Different data has different caching needs. Video metadata (rarely changes) gets long TTLs. Viewing history (changes with every watch) gets shorter TTLs with write-through updates.

---

## Amazon: Segmented Caching

Amazon's product catalog has millions of items, and different parts of a product page have different update frequencies.

**They don't cache "the product" as one blob.** They cache segments:

```
Product Details Cache (title, description, images)
- TTL: 24 hours
- Rarely changes
- High cache hit ratio

Price Cache
- TTL: 5 minutes
- Changes frequently (deals, dynamic pricing)
- Event-based invalidation when price changes

Inventory Cache
- TTL: 1 minute (or less)
- Needs near-real-time accuracy
- Write-through to keep in sync
- Critical for "only 3 left!" messaging

Review Cache
- TTL: 30 minutes
- Aggregated data (average rating, review count)
- Background refresh
- Slight staleness is acceptable
```

**Why segment?** If you cached the entire product as one object, you'd have to invalidate the whole thing every time the price changed. By segmenting, you only invalidate what actually changed.

---

## Facebook: TAO

Facebook's social graph (friends, likes, posts, comments) is massive and interconnected.

**TAO (The Associations and Objects)** is their custom caching layer:

- Caches objects (users, posts, photos) and associations (friendships, likes)
- Handles billions of queries per second
- 99.9%+ cache hit ratio

**How it works:**

```
Objects: Things with IDs
- User 123: {name: "John", ...}
- Post 456: {content: "Hello", ...}

Associations: Relationships between objects
- User 123 → friends → [User 456, User 789, ...]
- Post 456 → liked_by → [User 123, User 789, ...]
```

**Different TTLs for different data:**

| Data type | TTL | Why |
|-----------|-----|-----|
| Friend list | Long (hours) | Rarely changes |
| Like count | Short (minutes) | Changes frequently |
| Comments | Medium + event invalidation | Moderate change rate |
| Profile info | Long + event invalidation | Rarely changes |

**The key insight:** Social graph queries are highly repetitive. "Show me John's friends" gets asked millions of times. Caching these associations is extremely effective.

---

## Common patterns across all of them

1. **Multi-layer caching.** Client → CDN → Application cache → Database. Each layer catches requests before they hit the next.

2. **Segment by update frequency.** Don't cache everything together. Separate data that changes often from data that doesn't.

3. **Hybrid invalidation.** TTL as a safety net, plus event-based invalidation for important changes.

4. **Custom solutions at scale.** All of these companies built custom caching infrastructure (EVCache, TAO, etc.) because off-the-shelf solutions didn't meet their needs. You probably don't need to do this.

5. **Measure everything.** They all obsess over hit ratios, latency percentiles, and cache efficiency. You can't optimize what you don't measure.

---

## What this means for you

You're probably not operating at Twitter/Netflix scale. But the principles apply:

- **Start with Redis.** It handles most use cases well.
- **Segment your data.** Don't cache everything with the same TTL.
- **Use TTL + explicit invalidation.** Belt and suspenders.
- **Measure your hit ratio.** If it's below 80%, something's wrong.
- **Add layers only when needed.** Local cache in front of Redis, CDN for static content.

The fancy custom solutions come later, if ever. Most applications never need them.

---

[Next: Best Practices →](./11-best-practices.md)