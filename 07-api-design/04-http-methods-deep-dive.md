# HTTP Methods Deep Dive

[← Back to API Design](./README.md) | [← Previous: URL Design](./03-url-design.md)

---

HTTP methods (also called verbs) tell the server what operation to perform on a resource. Most developers know GET and POST, but there's more nuance to using them correctly than you might think.

Getting this right matters. Using the wrong method can break caching, cause security issues, and confuse developers using your API.

---

## The HTTP methods

| Method | Purpose | Safe | Idempotent | Has Request Body | Has Response Body |
|--------|---------|------|------------|------------------|-------------------|
| GET | Retrieve a resource | ✅ | ✅ | ❌ | ✅ |
| HEAD | Get headers only | ✅ | ✅ | ❌ | ❌ |
| POST | Create a resource / trigger action | ❌ | ❌ | ✅ | ✅ |
| PUT | Replace a resource | ❌ | ✅ | ✅ | ✅ |
| PATCH | Partially update a resource | ❌ | ❌* | ✅ | ✅ |
| DELETE | Remove a resource | ❌ | ✅ | ❌** | ✅ |
| OPTIONS | Get allowed methods | ✅ | ✅ | ❌ | ✅ |
| TRACE | Echo request (debugging) | ✅ | ✅ | ❌ | ✅ |

**Safe:** Doesn't modify server state. Can be cached, prefetched, etc.
**Idempotent:** Multiple identical requests have the same effect as one request.

*PATCH can be idempotent if designed carefully.
**DELETE technically can have a body, but it's discouraged.

---

## GET - Retrieve resources

GET retrieves a resource without modifying it. It's the most common method.

### Basic usage

```http
GET /users/123 HTTP/1.1
Host: api.example.com
Accept: application/json

HTTP/1.1 200 OK
Content-Type: application/json

{
  "id": "123",
  "name": "John Doe",
  "email": "john@example.com"
}
```

### GET for collections

```http
GET /users HTTP/1.1
Host: api.example.com

HTTP/1.1 200 OK
Content-Type: application/json

{
  "data": [
    {"id": "123", "name": "John"},
    {"id": "456", "name": "Jane"}
  ],
  "pagination": {
    "total": 100,
    "page": 1,
    "limit": 20
  }
}
```

### GET with query parameters

```http
GET /users?status=active&sort=-created_at&limit=10 HTTP/1.1
Host: api.example.com
```

### Key properties of GET

1. **Safe:** GET should never modify data. Ever. If your GET endpoint changes state, you're doing it wrong.

2. **Idempotent:** Calling GET multiple times returns the same result (assuming no other changes).

3. **Cacheable:** Responses can be cached by browsers, CDNs, and proxies.

4. **No request body:** GET requests shouldn't have a body. Some servers ignore it, others reject it.

### Common mistakes with GET

**Mistake 1: Using GET to modify data**
```
Bad:  GET /users/123/delete
      GET /orders/456?action=cancel

Good: DELETE /users/123
      POST /orders/456/cancellation
```

**Mistake 2: Putting sensitive data in URLs**
```
Bad:  GET /users?password=secret123
      GET /auth?token=abc123

Good: Use headers or POST body for sensitive data
```

URLs are logged, cached, and visible in browser history. Never put passwords, tokens, or PII in URLs.

**Mistake 3: Using GET for complex queries**
```
Bad:  GET /search?query=very-long-query-with-lots-of-parameters...
      (URLs have length limits, typically 2000-8000 characters)

Good: POST /search
      {
        "query": "...",
        "filters": {...}
      }
```

---

## POST - Create resources and trigger actions

POST submits data to be processed. It's used for creating resources and triggering actions.

### Creating a resource

```http
POST /users HTTP/1.1
Host: api.example.com
Content-Type: application/json

{
  "name": "John Doe",
  "email": "john@example.com"
}

HTTP/1.1 201 Created
Location: /users/123
Content-Type: application/json

{
  "id": "123",
  "name": "John Doe",
  "email": "john@example.com",
  "created_at": "2024-01-15T10:30:00Z"
}
```

Notice:
- Response is `201 Created`, not `200 OK`
- `Location` header points to the new resource
- Response includes the created resource with server-generated fields

### Triggering an action

```http
POST /users/123/password-reset HTTP/1.1
Host: api.example.com
Content-Type: application/json

{
  "email": "john@example.com"
}

HTTP/1.1 202 Accepted
Content-Type: application/json

{
  "message": "Password reset email sent",
  "expires_at": "2024-01-15T11:30:00Z"
}
```

### Key properties of POST

1. **Not safe:** POST modifies server state.

2. **Not idempotent:** Calling POST twice typically creates two resources.

3. **Not cacheable:** By default. Can be made cacheable with explicit headers.

4. **Has request body:** The body contains the data to process.

### Making POST idempotent

POST isn't naturally idempotent, but you can make it so with idempotency keys:

```http
POST /orders HTTP/1.1
Host: api.example.com
Content-Type: application/json
Idempotency-Key: order-abc-123-xyz

{
  "product_id": "456",
  "quantity": 1
}
```

The server stores the idempotency key and result. If the same key is sent again:
- If the original request succeeded, return the same response
- If the original request is still processing, return `409 Conflict`
- If the original request failed, allow retry

This is critical for payment APIs. You don't want to charge a customer twice because of a network timeout.

### When to use POST vs. PUT

| Scenario | Use |
|----------|-----|
| Server generates the ID | POST |
| Client provides the ID | PUT |
| Creating a new resource | POST (usually) |
| Replacing an existing resource | PUT |
| Partial update | PATCH (or POST) |
| Triggering an action | POST |

---

## PUT - Replace resources

PUT replaces an entire resource with the provided data. If the resource doesn't exist, it may create it.

### Replacing a resource

```http
PUT /users/123 HTTP/1.1
Host: api.example.com
Content-Type: application/json

{
  "name": "John Smith",
  "email": "johnsmith@example.com",
  "phone": "+1234567890"
}

HTTP/1.1 200 OK
Content-Type: application/json

{
  "id": "123",
  "name": "John Smith",
  "email": "johnsmith@example.com",
  "phone": "+1234567890",
  "updated_at": "2024-01-15T10:30:00Z"
}
```

### Key properties of PUT

1. **Not safe:** PUT modifies server state.

2. **Idempotent:** Calling PUT multiple times with the same data has the same effect as calling it once.

3. **Replaces entirely:** If you PUT with only `name`, other fields should be removed or set to null.

### PUT semantics: Replace, not update

This is important. PUT means "replace this resource with what I'm sending."

```http
Current state:
{
  "id": "123",
  "name": "John",
  "email": "john@example.com",
  "phone": "+1234567890"
}

PUT /users/123
{
  "name": "John Doe",
  "email": "johndoe@example.com"
}

Result (correct PUT semantics):
{
  "id": "123",
  "name": "John Doe",
  "email": "johndoe@example.com",
  "phone": null  ← Phone was removed!
}
```

If you want to update only some fields, use PATCH.

### PUT for creation (upsert)

PUT can create a resource if the client provides the ID:

```http
PUT /users/custom-id-123 HTTP/1.1
Host: api.example.com
Content-Type: application/json

{
  "name": "John Doe",
  "email": "john@example.com"
}

HTTP/1.1 201 Created  ← Created, not updated
Location: /users/custom-id-123
```

This is called "upsert" - update if exists, insert if not.

---

## PATCH - Partial updates

PATCH updates only the specified fields, leaving others unchanged.

### Basic PATCH

```http
PATCH /users/123 HTTP/1.1
Host: api.example.com
Content-Type: application/json

{
  "name": "John Smith"
}

HTTP/1.1 200 OK
Content-Type: application/json

{
  "id": "123",
  "name": "John Smith",      ← Updated
  "email": "john@example.com", ← Unchanged
  "phone": "+1234567890"       ← Unchanged
}
```

### PATCH formats

There are different ways to format PATCH requests:

**JSON Merge Patch (RFC 7396)**
```http
PATCH /users/123 HTTP/1.1
Content-Type: application/merge-patch+json

{
  "name": "John Smith",
  "phone": null  ← Explicitly set to null (removes it)
}
```

**JSON Patch (RFC 6902)**
```http
PATCH /users/123 HTTP/1.1
Content-Type: application/json-patch+json

[
  {"op": "replace", "path": "/name", "value": "John Smith"},
  {"op": "remove", "path": "/phone"},
  {"op": "add", "path": "/nickname", "value": "Johnny"}
]
```

JSON Patch is more powerful (supports operations like move, copy, test) but more complex. Most APIs use simple JSON or JSON Merge Patch.

### Key properties of PATCH

1. **Not safe:** PATCH modifies server state.

2. **Not idempotent (by default):** But can be designed to be idempotent.

3. **Partial update:** Only specified fields are changed.

### Making PATCH idempotent

PATCH can be idempotent if you design it carefully:

```
Idempotent:
PATCH /users/123 {"name": "John"}
→ Always sets name to "John"

Not idempotent:
PATCH /users/123 {"balance": {"increment": 10}}
→ Each call adds 10 to balance
```

### PUT vs. PATCH

| Aspect | PUT | PATCH |
|--------|-----|-------|
| Semantics | Replace entire resource | Update partial resource |
| Missing fields | Removed/nulled | Unchanged |
| Idempotent | Yes | Depends on implementation |
| Use case | Full updates | Partial updates |

**My recommendation:** Use PATCH for updates in most cases. It's more intuitive and less error-prone.

---

## DELETE - Remove resources

DELETE removes a resource.

### Basic DELETE

```http
DELETE /users/123 HTTP/1.1
Host: api.example.com

HTTP/1.1 204 No Content
```

### DELETE with response body

Some APIs return the deleted resource:

```http
DELETE /users/123 HTTP/1.1
Host: api.example.com

HTTP/1.1 200 OK
Content-Type: application/json

{
  "id": "123",
  "name": "John Doe",
  "deleted_at": "2024-01-15T10:30:00Z"
}
```

### Key properties of DELETE

1. **Not safe:** DELETE modifies server state.

2. **Idempotent:** Deleting the same resource twice has the same effect as deleting it once.

3. **No request body (usually):** The HTTP spec allows it, but many servers don't support it.

### Idempotency of DELETE

DELETE is idempotent, but what should happen on the second call?

**Option 1: Return 404**
```http
DELETE /users/123  → 204 No Content
DELETE /users/123  → 404 Not Found
```

**Option 2: Return 204 (or 200)**
```http
DELETE /users/123  → 204 No Content
DELETE /users/123  → 204 No Content (already deleted, but that's fine)
```

Both are valid. Option 2 is more "idempotent" in spirit. Option 1 is more informative.

### Soft delete vs. hard delete

**Hard delete:** Resource is permanently removed.
```http
DELETE /users/123
→ User is gone forever
```

**Soft delete:** Resource is marked as deleted but retained.
```http
DELETE /users/123
→ User is marked deleted, can be restored

GET /users/123
→ 404 Not Found (or 410 Gone)

GET /users/123?include_deleted=true
→ 200 OK with deleted user
```

Soft delete is common for:
- Audit requirements
- Accidental deletion recovery
- Data retention policies

---

## HEAD - Get headers only

HEAD is identical to GET but returns only headers, no body. Useful for checking if a resource exists or getting metadata.

```http
HEAD /users/123 HTTP/1.1
Host: api.example.com

HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 256
Last-Modified: Mon, 15 Jan 2024 10:30:00 GMT
ETag: "abc123"
```

### Use cases for HEAD

1. **Check if resource exists**
```http
HEAD /users/123 → 200 OK (exists)
HEAD /users/999 → 404 Not Found (doesn't exist)
```

2. **Get resource metadata**
```http
HEAD /files/large-file.zip
→ Content-Length: 1073741824 (1GB)
```

3. **Check for updates (caching)**
```http
HEAD /users/123
If-None-Match: "abc123"
→ 304 Not Modified (cache is still valid)
```

---

## OPTIONS - Get allowed methods

OPTIONS returns the HTTP methods allowed for a resource. It's also used for CORS preflight requests.

```http
OPTIONS /users/123 HTTP/1.1
Host: api.example.com

HTTP/1.1 200 OK
Allow: GET, PUT, PATCH, DELETE, HEAD, OPTIONS
```

### CORS preflight

Browsers send OPTIONS requests before cross-origin requests to check if they're allowed:

```http
OPTIONS /users HTTP/1.1
Host: api.example.com
Origin: https://webapp.example.com
Access-Control-Request-Method: POST
Access-Control-Request-Headers: Content-Type, Authorization

HTTP/1.1 200 OK
Access-Control-Allow-Origin: https://webapp.example.com
Access-Control-Allow-Methods: GET, POST, PUT, DELETE
Access-Control-Allow-Headers: Content-Type, Authorization
Access-Control-Max-Age: 86400
```

---

## Method selection decision tree

```
What are you trying to do?

├── Read data?
│   ├── Need the body? → GET
│   └── Just checking existence/metadata? → HEAD
│
├── Create something?
│   ├── Server generates ID? → POST
│   └── Client provides ID? → PUT (or POST)
│
├── Update something?
│   ├── Replace entirely? → PUT
│   └── Update partially? → PATCH
│
├── Delete something? → DELETE
│
├── Trigger an action? → POST
│
└── Check what's allowed? → OPTIONS
```

---

## Handling non-CRUD operations

Not everything fits neatly into CRUD. Here's how to handle special cases:

### State transitions

```http
# Option 1: Action as sub-resource
POST /orders/123/cancel
POST /users/456/activate
POST /documents/789/publish

# Option 2: PATCH the state
PATCH /orders/123
{"status": "cancelled"}

# Option 3: PUT to a state resource
PUT /orders/123/status
{"status": "cancelled"}
```

### Bulk operations

```http
# Option 1: Batch endpoint
POST /users/batch
{
  "operations": [
    {"method": "POST", "body": {"name": "John"}},
    {"method": "DELETE", "id": "123"}
  ]
}

# Option 2: Query parameter
DELETE /users?ids=123,456,789

# Option 3: Request body (non-standard for DELETE)
DELETE /users
{"ids": ["123", "456", "789"]}
```

### Search with complex criteria

```http
# Simple search: GET with query params
GET /products?category=electronics&price_min=100&price_max=500

# Complex search: POST (when query is too complex for URL)
POST /products/search
{
  "query": "laptop",
  "filters": {
    "category": ["electronics", "computers"],
    "price": {"min": 100, "max": 500},
    "attributes": {
      "brand": ["Apple", "Dell"],
      "ram": {"gte": 8}
    }
  },
  "sort": ["-rating", "price"]
}
```

### Long-running operations

```http
# Start the operation
POST /reports/generate
{
  "type": "sales",
  "date_range": {"start": "2024-01-01", "end": "2024-01-31"}
}

HTTP/1.1 202 Accepted
Location: /jobs/abc123

{
  "job_id": "abc123",
  "status": "pending",
  "status_url": "/jobs/abc123"
}

# Check status
GET /jobs/abc123

HTTP/1.1 200 OK
{
  "job_id": "abc123",
  "status": "completed",
  "result_url": "/reports/xyz789"
}

# Get result
GET /reports/xyz789
```

---

## Method safety and caching

### Safe methods and caching

Safe methods (GET, HEAD, OPTIONS) can be cached because they don't modify state:

```http
GET /products/123 HTTP/1.1
Host: api.example.com

HTTP/1.1 200 OK
Cache-Control: max-age=3600
ETag: "abc123"

{...}
```

### Unsafe methods and cache invalidation

Unsafe methods (POST, PUT, PATCH, DELETE) should invalidate related caches:

```http
PUT /products/123 HTTP/1.1
Host: api.example.com

{...}

HTTP/1.1 200 OK
Cache-Control: no-cache

# Client should invalidate cached GET /products/123
```

---

## Common mistakes

### 1. Using GET for mutations

```
Bad:  GET /users/123/delete
      GET /transfer?from=A&to=B&amount=100

Good: DELETE /users/123
      POST /transfers {"from": "A", "to": "B", "amount": 100}
```

### 2. Using POST for everything

```
Bad:  POST /users/123/get
      POST /users/123/update
      POST /users/123/delete

Good: GET /users/123
      PATCH /users/123
      DELETE /users/123
```

### 3. Confusing PUT and PATCH

```
PUT /users/123 {"name": "John"}
→ Should remove email, phone, etc. (replace semantics)

PATCH /users/123 {"name": "John"}
→ Should only update name (partial update)
```

### 4. Not handling idempotency

```
Bad:  POST /orders (no idempotency key)
      → Network timeout, client retries, two orders created

Good: POST /orders
      Idempotency-Key: order-123
      → Retry is safe
```

### 5. Returning wrong status codes

```
Bad:  DELETE /users/123 → 200 OK {"success": true}
Good: DELETE /users/123 → 204 No Content

Bad:  POST /users → 200 OK {...}
Good: POST /users → 201 Created {...}
```

---

## Key takeaways

1. **GET is for reading.** Never use it to modify data.

2. **POST is for creating and actions.** Use idempotency keys for critical operations.

3. **PUT replaces entirely.** Missing fields are removed.

4. **PATCH updates partially.** Only specified fields change.

5. **DELETE is idempotent.** Deleting twice should be safe.

6. **Use the right method.** It affects caching, security, and developer expectations.

7. **Idempotency matters.** Design for network failures and retries.

---

[Next: Request & Response Design →](./05-request-response-design.md)