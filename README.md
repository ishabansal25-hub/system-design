# System Design Notes

Hey! 👋 I'm building this repo to document what I learn about system design. It's mostly for myself, but hopefully others find it useful too.

## What's here right now?

**Only caching is done so far.** I went pretty deep on it - 12 files covering everything from basics to distributed caching, plus 7 real scenarios that actually come up in production.

| What's Ready | Status |
|--------------|--------|
| [Caching](./04-caching/) | ✅ Done |
| Everything else | 🚧 Not yet |

---

## Caching (the good stuff)

I spent a lot of time on this because caching is everywhere and it's easy to mess up. Here's what's covered:

**Concepts:**
- What caching actually is (and isn't)
- Different types - browser, CDN, app-level, database
- Strategies like Cache-Aside, Write-Through, etc.
- Eviction policies (LRU, LFU, TTL)
- Redis vs Memcached - when to use what
- Distributed caching and consistent hashing
- Cache invalidation (the hard part)
- Stampede prevention

**Real scenarios I wrote up:**
1. Your hot cache key expires and 50k requests hit the DB at once
2. "Why don't we just cache everything?" (spoiler: bad idea)
3. Keeping cache and DB in sync without race conditions
4. Local cache vs Redis - tradeoffs
5. Cold start after deployment
6. Your Redis bill is $20k/month, now what?
7. Users keep requesting IDs that don't exist

Each scenario has the problem, why it matters, multiple solutions with tradeoffs, and how I'd actually explain it.

� **[Start here](./04-caching/)**

---

## What I'm planning to add

Not in any particular order, just topics I want to cover eventually:

- **Fundamentals** - CAP theorem, ACID vs BASE, networking basics
- **Scalability** - Load balancing, sharding, replication
- **Databases** - SQL vs NoSQL, when to use what, indexing
- **Message Queues** - Kafka, RabbitMQ, event-driven stuff
- **Microservices** - Service discovery, circuit breakers, sagas
- **API Design** - REST, GraphQL, gRPC, rate limiting
- **Security** - Auth, encryption, common vulnerabilities
- **Observability** - Logging, metrics, tracing
- **AI/ML Systems** - LLM serving, RAG, vector DBs
- **Case Studies** - Twitter, Netflix, Uber, etc.

---

## How I'm organizing this

```
system-design/
├── 04-caching/          ← start here, it's done
├── 01-fundamentals/     ← empty for now
├── 02-scalability/      ← empty for now
├── ...
└── 11-case-studies/     ← empty for now
```

---

## Resources I've found helpful

**Books:**
- "Designing Data-Intensive Applications" - Martin Kleppmann (the bible)
- "System Design Interview" - Alex Xu (good for structured prep)
- "Building Microservices" - Sam Newman

**Online:**
- [System Design Primer](https://github.com/donnemartin/system-design-primer)
- [AWS Architecture Center](https://aws.amazon.com/architecture/)

---

## Why I'm doing this

Honestly? I learn better when I write things down. And I kept forgetting caching concepts, so I decided to document everything properly. If it helps someone else, that's a bonus.

Feel free to open issues or PRs if you spot mistakes or want to add something.

---

*Last updated: March 2026*