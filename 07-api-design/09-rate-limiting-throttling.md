# Rate Limiting & Throttling

[← Back to API Design](./README.md) | [← Previous: Authentication & Authorization](./08-authentication-authorization.md)

---

Rate limiting is how you protect your API from being overwhelmed. Without it, a single misbehaving client can take down your entire service, whether intentionally (DDoS attack) or accidentally (buggy code in a loop).

I've seen APIs go down because someone wrote a script that made 10,000 requests per second. Don't let that happen to you.

---

## Why rate limiting matters

### Protection against abuse

```
Without rate limiting:
- Malicious actors can DDoS your API
- Buggy clients can accidentally overwhelm you
- Scrapers can steal your data
- Competitors can increase your infrastructure costs

With rate limiting:
- Each client gets fair access
- Your infrastructure stays healthy
- Costs remain predictable
- You can identify and block bad actors
```

### Fair resource allocation

```
Without limits:
- One heavy user consumes 90% of resources
- Other users experience slow responses or timeouts

With limits:
- Each user gets a fair share
- Consistent experience for everyone
```

---

## Rate limiting vs. throttling

These terms are often used interchangeably, but there's a subtle difference:

| Concept | What it does |
|---------|--------------|
| **Rate Limiting** | Rejects requests that exceed the limit |
| **Throttling** | Slows down requests that exceed the limit |

**Rate limiting:**
```
Request 101 (over limit) → 429 Too Many Requests (rejected)
```

**Throttling:**
```
Request 101 (over limit) → Queued, processed after delay
```

Most APIs use rate limiting (rejection) because throttling can lead to unpredictable latencies.

---

## Rate limiting algorithms

### 1. Fixed Window

Count requests in fixed time windows (e.g., per minute).

```
Window: 12:00:00 - 12:00:59
Limit: 100 requests

12:00:15 - Request 1   ✓
12:00:30 - Request 50  ✓
12:00:45 - Request 100 ✓
12:00:50 - Request 101 ✗ (rejected)
12:01:00 - Request 1   ✓ (new window)
```

**Implementation:**
```python
def is_allowed(user_id, limit=100, window=60):
    key = f"rate:{user_id}:{current_minute()}"
    count = redis.incr(key)
    if count == 1:
        redis.expire(key, window)
    return count <= limit
```

**Pros:**
- Simple to implement
- Low memory usage

**Cons:**
- Burst at window boundaries (can get 200 requests in 2 seconds if timed right)

```
Problem: Window boundary burst
11:59:59 - 100 requests ✓
12:00:01 - 100 requests ✓
= 200 requests in 2 seconds!
```

### 2. Sliding Window Log

Track timestamps of all requests, count those within the window.

```
Window: Last 60 seconds
Limit: 100 requests

Timestamps: [11:59:30, 11:59:45, 12:00:10, 12:00:15, ...]

At 12:00:30:
- Remove timestamps older than 11:59:30
- Count remaining timestamps
- Allow if count < 100
```

**Implementation:**
```python
def is_allowed(user_id, limit=100, window=60):
    now = time.time()
    key = f"rate:{user_id}"
    
    # Remove old entries
    redis.zremrangebyscore(key, 0, now - window)
    
    # Count current entries
    count = redis.zcard(key)
    
    if count < limit:
        redis.zadd(key, {str(now): now})
        redis.expire(key, window)
        return True
    return False
```

**Pros:**
- Accurate, no boundary issues
- Smooth rate limiting

**Cons:**
- Higher memory usage (stores all timestamps)
- More expensive operations

### 3. Sliding Window Counter

Hybrid approach: weighted average of current and previous window.

```
Previous window (11:59): 80 requests
Current window (12:00): 20 requests
Current position: 30 seconds into window (50%)

Weighted count = 80 * 0.5 + 20 = 60 requests
Limit: 100
Result: Allowed (60 < 100)
```

**Implementation:**
```python
def is_allowed(user_id, limit=100, window=60):
    now = time.time()
    current_window = int(now / window)
    previous_window = current_window - 1
    position_in_window = (now % window) / window
    
    current_count = redis.get(f"rate:{user_id}:{current_window}") or 0
    previous_count = redis.get(f"rate:{user_id}:{previous_window}") or 0
    
    weighted_count = previous_count * (1 - position_in_window) + current_count
    
    if weighted_count < limit:
        redis.incr(f"rate:{user_id}:{current_window}")
        return True
    return False
```

**Pros:**
- Smooths out boundary issues
- Low memory usage
- Good balance of accuracy and efficiency

**Cons:**
- Slightly more complex
- Approximation, not exact

### 4. Token Bucket

Tokens are added at a fixed rate. Each request consumes a token.

```
Bucket capacity: 100 tokens
Refill rate: 10 tokens/second

Initial: 100 tokens
Request 1: 99 tokens
Request 2: 98 tokens
...
Request 100: 0 tokens
Request 101: Rejected (no tokens)

After 1 second: 10 tokens added
Request 102: 9 tokens ✓
```

**Implementation:**
```python
def is_allowed(user_id, capacity=100, refill_rate=10):
    now = time.time()
    key = f"bucket:{user_id}"
    
    # Get current state
    data = redis.hgetall(key)
    tokens = float(data.get('tokens', capacity))
    last_refill = float(data.get('last_refill', now))
    
    # Refill tokens
    elapsed = now - last_refill
    tokens = min(capacity, tokens + elapsed * refill_rate)
    
    if tokens >= 1:
        tokens -= 1
        redis.hset(key, mapping={'tokens': tokens, 'last_refill': now})
        return True
    return False
```

**Pros:**
- Allows bursts up to bucket capacity
- Smooth rate limiting over time
- Flexible configuration

**Cons:**
- More complex to implement
- Requires storing state per user

### 5. Leaky Bucket

Requests are added to a queue that drains at a fixed rate.

```
Bucket capacity: 100 requests
Drain rate: 10 requests/second

Requests enter the bucket
Bucket drains at constant rate
If bucket overflows, requests are rejected
```

**Pros:**
- Smooths out bursts
- Constant output rate

**Cons:**
- Can add latency (requests wait in queue)
- More complex to implement

### Algorithm comparison

| Algorithm | Burst handling | Memory | Accuracy | Complexity |
|-----------|---------------|--------|----------|------------|
| Fixed Window | Poor | Low | Low | Simple |
| Sliding Log | Good | High | High | Medium |
| Sliding Counter | Good | Low | Medium | Medium |
| Token Bucket | Configurable | Medium | High | Medium |
| Leaky Bucket | Smoothed | Medium | High | Complex |

**My recommendation:** Token Bucket for most cases. It's flexible, accurate, and handles bursts well.

---

## Rate limit headers

Always tell clients about their rate limit status:

```http
HTTP/1.1 200 OK
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 95
X-RateLimit-Reset: 1705312200

{
  "data": {...}
}
```

| Header | Meaning |
|--------|---------|
| `X-RateLimit-Limit` | Maximum requests allowed in window |
| `X-RateLimit-Remaining` | Requests remaining in current window |
| `X-RateLimit-Reset` | Unix timestamp when limit resets |

Some APIs use different header names:
```http
RateLimit-Limit: 100
RateLimit-Remaining: 95
RateLimit-Reset: 60

# Or GitHub style
X-RateLimit-Limit: 5000
X-RateLimit-Remaining: 4999
X-RateLimit-Reset: 1372700873
```

### Rate limit exceeded response

```http
HTTP/1.1 429 Too Many Requests
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1705312200
Retry-After: 60
Content-Type: application/json

{
  "error": {
    "code": "RATE_LIMITED",
    "message": "Too many requests. Please retry after 60 seconds.",
    "retry_after": 60
  }
}
```

Always include `Retry-After` header to tell clients when to retry.

---

## Rate limiting strategies

### By client/API key

```python
def get_rate_limit_key(request):
    return f"rate:{request.api_key}"

# Limits per API key
# Free tier: 100 requests/minute
# Pro tier: 1000 requests/minute
```

### By user

```python
def get_rate_limit_key(request):
    return f"rate:{request.user_id}"
```

### By IP address

```python
def get_rate_limit_key(request):
    return f"rate:{request.client_ip}"
```

**Caution:** Multiple users behind NAT share the same IP.

### By endpoint

```python
def get_rate_limit_key(request):
    return f"rate:{request.api_key}:{request.endpoint}"

# Different limits for different endpoints
# GET /users: 1000/minute
# POST /users: 100/minute
# POST /payments: 10/minute
```

### Combined

```python
def get_rate_limit_key(request):
    # Global limit per API key
    global_key = f"rate:global:{request.api_key}"
    
    # Per-endpoint limit
    endpoint_key = f"rate:endpoint:{request.api_key}:{request.endpoint}"
    
    return [global_key, endpoint_key]

# Check both limits
# Global: 10000 requests/hour
# POST /payments: 100 requests/hour
```

---

## Tiered rate limits

Different limits for different user tiers:

```python
RATE_LIMITS = {
    'free': {
        'requests_per_minute': 60,
        'requests_per_day': 1000,
    },
    'basic': {
        'requests_per_minute': 300,
        'requests_per_day': 10000,
    },
    'pro': {
        'requests_per_minute': 1000,
        'requests_per_day': 100000,
    },
    'enterprise': {
        'requests_per_minute': 10000,
        'requests_per_day': 1000000,
    },
}

def get_rate_limit(user):
    tier = user.subscription_tier
    return RATE_LIMITS.get(tier, RATE_LIMITS['free'])
```

### Communicating limits

```http
GET /rate-limit HTTP/1.1
Authorization: Bearer eyJ...

HTTP/1.1 200 OK
{
  "tier": "pro",
  "limits": {
    "requests_per_minute": 1000,
    "requests_per_day": 100000
  },
  "current_usage": {
    "minute": 45,
    "day": 5230
  },
  "resets": {
    "minute": "2024-01-15T10:31:00Z",
    "day": "2024-01-16T00:00:00Z"
  }
}
```

---

## Handling rate limit exceeded

### Client-side handling

```javascript
async function makeRequest(url, options, retries = 3) {
  const response = await fetch(url, options);
  
  if (response.status === 429 && retries > 0) {
    const retryAfter = response.headers.get('Retry-After') || 60;
    console.log(`Rate limited. Retrying after ${retryAfter} seconds`);
    
    await sleep(retryAfter * 1000);
    return makeRequest(url, options, retries - 1);
  }
  
  return response;
}
```

### Exponential backoff

```javascript
async function makeRequestWithBackoff(url, options, attempt = 1) {
  const response = await fetch(url, options);
  
  if (response.status === 429 && attempt <= 5) {
    // Exponential backoff with jitter
    const baseDelay = Math.pow(2, attempt) * 1000; // 2s, 4s, 8s, 16s, 32s
    const jitter = Math.random() * 1000;
    const delay = baseDelay + jitter;
    
    console.log(`Rate limited. Retrying in ${delay}ms (attempt ${attempt})`);
    await sleep(delay);
    return makeRequestWithBackoff(url, options, attempt + 1);
  }
  
  return response;
}
```

---

## Distributed rate limiting

When you have multiple API servers, you need shared state.

### Redis-based

```python
import redis

redis_client = redis.Redis(host='localhost', port=6379)

def is_allowed(user_id, limit=100, window=60):
    key = f"rate:{user_id}"
    
    # Atomic increment and check
    pipe = redis_client.pipeline()
    pipe.incr(key)
    pipe.expire(key, window)
    results = pipe.execute()
    
    count = results[0]
    return count <= limit
```

### Redis with Lua script (atomic)

```lua
-- rate_limit.lua
local key = KEYS[1]
local limit = tonumber(ARGV[1])
local window = tonumber(ARGV[2])

local current = redis.call('INCR', key)
if current == 1 then
    redis.call('EXPIRE', key, window)
end

if current > limit then
    return 0  -- Rejected
else
    return 1  -- Allowed
end
```

```python
def is_allowed(user_id, limit=100, window=60):
    key = f"rate:{user_id}"
    result = redis_client.eval(RATE_LIMIT_SCRIPT, 1, key, limit, window)
    return result == 1
```

### Considerations for distributed systems

```
1. Clock synchronization
   - Use Redis server time, not local time
   - Or use logical timestamps

2. Network latency
   - Rate limit check adds latency
   - Consider local caching with periodic sync

3. Failure handling
   - What if Redis is down?
   - Fail open (allow) or fail closed (reject)?

4. Consistency
   - Eventual consistency is usually acceptable
   - Strict consistency requires more coordination
```

---

## Advanced patterns

### Adaptive rate limiting

Adjust limits based on system load:

```python
def get_dynamic_limit(base_limit):
    cpu_usage = get_cpu_usage()
    
    if cpu_usage > 90:
        return base_limit * 0.5  # Reduce to 50%
    elif cpu_usage > 70:
        return base_limit * 0.75  # Reduce to 75%
    else:
        return base_limit
```

### Priority queues

Different priorities for different request types:

```python
PRIORITIES = {
    'critical': {'limit': 1000, 'weight': 1},
    'normal': {'limit': 100, 'weight': 2},
    'background': {'limit': 10, 'weight': 5},
}

def is_allowed(user_id, priority='normal'):
    config = PRIORITIES[priority]
    # Higher weight = more tokens consumed
    return token_bucket.consume(user_id, config['weight'])
```

### Cost-based rate limiting

Different endpoints have different costs:

```python
ENDPOINT_COSTS = {
    'GET /users': 1,
    'GET /users/{id}': 1,
    'POST /users': 5,
    'GET /reports': 10,
    'POST /bulk-import': 100,
}

def is_allowed(user_id, endpoint, limit=1000):
    cost = ENDPOINT_COSTS.get(endpoint, 1)
    return token_bucket.consume(user_id, cost, limit)
```

### Quota management

Long-term quotas in addition to rate limits:

```python
def check_quota(user_id):
    # Rate limit: 100 requests/minute
    if not rate_limiter.is_allowed(user_id, 100, 60):
        return False, "Rate limit exceeded"
    
    # Daily quota: 10,000 requests/day
    daily_count = get_daily_count(user_id)
    if daily_count >= 10000:
        return False, "Daily quota exceeded"
    
    # Monthly quota: 100,000 requests/month
    monthly_count = get_monthly_count(user_id)
    if monthly_count >= 100000:
        return False, "Monthly quota exceeded"
    
    return True, None
```

---

## Real-world examples

### GitHub

```
Authenticated: 5,000 requests/hour
Unauthenticated: 60 requests/hour

Headers:
X-RateLimit-Limit: 5000
X-RateLimit-Remaining: 4999
X-RateLimit-Reset: 1372700873
X-RateLimit-Used: 1
X-RateLimit-Resource: core
```

### Stripe

```
Live mode: 100 requests/second
Test mode: 25 requests/second

Headers:
RateLimit-Limit: 100
RateLimit-Remaining: 99
RateLimit-Reset: 1
```

### Twitter/X

```
App-level: 300 requests/15 minutes
User-level: 900 requests/15 minutes

Different limits per endpoint
```

---

## Best practices

### 1. Always return rate limit headers

```http
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 95
X-RateLimit-Reset: 1705312200
```

### 2. Use 429 status code

```http
HTTP/1.1 429 Too Many Requests
Retry-After: 60
```

### 3. Provide clear error messages

```json
{
  "error": {
    "code": "RATE_LIMITED",
    "message": "You have exceeded the rate limit of 100 requests per minute",
    "retry_after": 60,
    "documentation_url": "https://api.example.com/docs/rate-limiting"
  }
}
```

### 4. Document your limits

```markdown
## Rate Limits

| Tier | Requests/minute | Requests/day |
|------|-----------------|--------------|
| Free | 60 | 1,000 |
| Pro | 600 | 50,000 |
| Enterprise | Custom | Custom |

Rate limit headers are included in every response.
```

### 5. Allow bursts

```
Token bucket with:
- Capacity: 100 (allows burst of 100)
- Refill rate: 10/second (sustained rate)
```

### 6. Have different limits for different operations

```
GET requests: 1000/minute
POST requests: 100/minute
DELETE requests: 50/minute
```

### 7. Monitor and alert

```
Track:
- Rate limit hits per user
- 429 responses over time
- Users consistently hitting limits

Alert when:
- Unusual spike in rate limit hits
- Single user consuming disproportionate resources
```

---

## Key takeaways

1. **Rate limiting is essential.** Protect your API from abuse and ensure fair access.

2. **Use Token Bucket** for most cases. It's flexible and handles bursts well.

3. **Always return rate limit headers.** Clients need to know their limits.

4. **Use 429 status code** with `Retry-After` header.

5. **Different limits for different tiers.** Monetize your API appropriately.

6. **Distributed rate limiting** requires shared state (Redis).

7. **Monitor and adjust.** Rate limits should evolve with your API usage.

---

[Next: Pagination, Filtering & Sorting →](./10-pagination-filtering-sorting.md)