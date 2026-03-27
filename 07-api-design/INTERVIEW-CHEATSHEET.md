# API Design Interview Cheatsheet

[← Back to API Design](./README.md)

---

Quick reference for API design interviews. Use this to refresh your memory before interviews or as a quick lookup during system design discussions.

---

## Quick Reference Table

| Topic | Key Points |
|-------|------------|
| REST | Resources, HTTP methods, stateless, uniform interface |
| HTTP Methods | GET (read), POST (create), PUT (replace), PATCH (update), DELETE (remove) |
| Status Codes | 2xx success, 4xx client error, 5xx server error |
| Auth | API keys, JWT, OAuth 2.0, sessions |
| Rate Limiting | Token bucket, sliding window, fixed window |
| Pagination | Offset (simple), Cursor (scalable), Time-based (logs) |
| Versioning | URL path (/v1/), header, query param |

---

## REST Fundamentals

### Six Constraints
1. **Client-Server** - Separation of concerns
2. **Stateless** - No session state on server
3. **Cacheable** - Responses must define cacheability
4. **Uniform Interface** - Consistent resource identification
5. **Layered System** - Client can't tell if connected directly
6. **Code on Demand** (optional) - Server can send executable code

### Resource Naming
```
✓ /users                    (collection)
✓ /users/123                (specific resource)
✓ /users/123/orders         (sub-resource)
✓ /users/123/orders/456     (specific sub-resource)

✗ /getUsers
✗ /user/create
✗ /deleteUser/123
```

---

## HTTP Methods

| Method | Purpose | Safe | Idempotent | Body |
|--------|---------|------|------------|------|
| GET | Read | ✅ | ✅ | ❌ |
| POST | Create | ❌ | ❌ | ✅ |
| PUT | Replace | ❌ | ✅ | ✅ |
| PATCH | Update | ❌ | ❌* | ✅ |
| DELETE | Remove | ❌ | ✅ | ❌ |

**Safe:** Doesn't modify state
**Idempotent:** Multiple calls = same result

---

## Status Codes

### Success (2xx)
- **200 OK** - Success with body
- **201 Created** - Resource created
- **202 Accepted** - Async processing started
- **204 No Content** - Success, no body

### Client Errors (4xx)
- **400 Bad Request** - Malformed request
- **401 Unauthorized** - Not authenticated
- **403 Forbidden** - Not authorized
- **404 Not Found** - Resource doesn't exist
- **409 Conflict** - Resource conflict
- **422 Unprocessable** - Validation error
- **429 Too Many Requests** - Rate limited

### Server Errors (5xx)
- **500 Internal Error** - Server error
- **502 Bad Gateway** - Upstream error
- **503 Unavailable** - Temporarily down
- **504 Gateway Timeout** - Upstream timeout

---

## Authentication Methods

### API Keys
```http
X-API-Key: sk_live_abc123
```
- Simple, good for server-to-server
- No expiration (unless built)
- Can't identify users

### JWT (JSON Web Token)
```http
Authorization: Bearer eyJhbGc...
```
- Stateless, contains claims
- Self-contained user info
- Can't revoke before expiration

### OAuth 2.0 Flows
| Flow | Use Case |
|------|----------|
| Authorization Code | Server-side apps |
| Auth Code + PKCE | Mobile/SPA |
| Client Credentials | Machine-to-machine |

---

## Rate Limiting Algorithms

### Token Bucket
- Tokens added at fixed rate
- Requests consume tokens
- Allows bursts up to bucket size

### Sliding Window
- Tracks requests in rolling window
- More accurate than fixed window
- Higher memory usage

### Fixed Window
- Count requests per time window
- Simple but has boundary issues
- Can allow 2x burst at boundaries

### Headers
```http
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 95
X-RateLimit-Reset: 1705312200
Retry-After: 60
```

---

## Pagination

### Offset-based
```http
GET /users?page=2&limit=20
GET /users?offset=20&limit=20
```
- Simple, can jump to any page
- Slow for large offsets
- Inconsistent with data changes

### Cursor-based
```http
GET /users?cursor=eyJpZCI6MTIzfQ&limit=20
```
- Fast at any position
- Consistent results
- Can't jump to arbitrary page

### When to use
- **Offset:** Small datasets, admin UIs
- **Cursor:** Large datasets, feeds, real-time

---

## Versioning Strategies

### URL Path (Recommended)
```http
GET /v1/users
GET /v2/users
```

### Header
```http
X-API-Version: 2
Accept: application/vnd.api.v2+json
```

### Query Parameter
```http
GET /users?version=2
```

### Breaking vs Non-breaking
| Breaking | Non-breaking |
|----------|--------------|
| Remove field | Add field |
| Rename field | Add endpoint |
| Change type | Add optional param |
| Remove endpoint | Fix bugs |

---

## Error Response Format

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Request validation failed",
    "details": [
      {
        "field": "email",
        "message": "Must be valid email"
      }
    ],
    "request_id": "req-123"
  }
}
```

---

## API Gateway Responsibilities

1. **Routing** - Direct requests to services
2. **Authentication** - Verify identity
3. **Rate Limiting** - Protect services
4. **Load Balancing** - Distribute traffic
5. **Caching** - Reduce backend load
6. **Transformation** - Modify requests/responses
7. **Monitoring** - Centralized logging

---

## REST vs GraphQL vs gRPC

| Aspect | REST | GraphQL | gRPC |
|--------|------|---------|------|
| Protocol | HTTP | HTTP | HTTP/2 |
| Format | JSON | JSON | Protobuf |
| Contract | OpenAPI | Schema | .proto |
| Flexibility | Fixed | Client-defined | Fixed |
| Performance | Good | Good | Excellent |
| Browser | Native | Native | Needs proxy |
| Use case | Public APIs | Complex UIs | Microservices |

---

## Common Interview Questions

### "Design an API for X"

**Framework:**
1. Clarify requirements
2. Identify resources
3. Define endpoints
4. Design request/response
5. Handle errors
6. Consider auth, rate limiting
7. Plan for scale

### "How would you handle..."

**Authentication:**
- JWT for stateless
- OAuth for third-party
- API keys for server-to-server

**Rate Limiting:**
- Token bucket for flexibility
- Per-user and per-endpoint limits
- Return 429 with Retry-After

**Versioning:**
- URL path for visibility
- Deprecation headers
- Migration guides

**Large Responses:**
- Pagination (cursor for scale)
- Field selection
- Compression

---

## Design Checklist

### Before Building
- [ ] Resources identified
- [ ] Endpoints defined
- [ ] Auth strategy chosen
- [ ] Rate limits planned
- [ ] Versioning strategy set

### For Each Endpoint
- [ ] Correct HTTP method
- [ ] Proper status codes
- [ ] Request validation
- [ ] Error handling
- [ ] Pagination (if collection)

### For Production
- [ ] Documentation complete
- [ ] Rate limiting implemented
- [ ] Monitoring in place
- [ ] Caching strategy
- [ ] Security reviewed

---

## Quick Formulas

### Rate Limit Calculation
```
Requests per second = Bucket capacity / Refill interval
Burst capacity = Bucket size
```

### Pagination
```
Offset = (page - 1) × limit
Total pages = ceil(total / limit)
```

### API Latency Budget
```
Total = Network + Gateway + Service + Database
Target: P99 < 200ms for reads
```

---

## Red Flags in API Design

❌ Verbs in URLs (`/getUser`, `/createOrder`)
❌ 200 OK for errors
❌ No versioning
❌ No rate limiting
❌ Exposing internal IDs
❌ Inconsistent naming
❌ No pagination
❌ Poor error messages

---

## Interview Tips

1. **Start with requirements** - Ask clarifying questions
2. **Think out loud** - Explain your reasoning
3. **Consider tradeoffs** - No perfect solution
4. **Mention scale** - How does it handle growth?
5. **Security first** - Auth, validation, rate limiting
6. **Be practical** - Real-world constraints matter

---

## Further Reading

- [REST Fundamentals](./02-rest-fundamentals.md)
- [HTTP Methods Deep Dive](./04-http-methods-deep-dive.md)
- [Status Codes & Errors](./06-status-codes-and-errors.md)
- [Authentication & Authorization](./08-authentication-authorization.md)
- [Rate Limiting](./09-rate-limiting-throttling.md)
- [Pagination](./10-pagination-filtering-sorting.md)
- [Real-World Examples](./15-real-world-api-examples.md)

---

*Good luck with your interview!* 🚀