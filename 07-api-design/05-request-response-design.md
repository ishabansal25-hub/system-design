# Request & Response Design

[← Back to API Design](./README.md) | [← Previous: HTTP Methods Deep Dive](./04-http-methods-deep-dive.md)

---

The request and response are where the rubber meets the road in API design. You can have perfect URLs and use the right HTTP methods, but if your payloads are confusing, inconsistent, or missing critical information, developers will hate your API.

I've seen APIs where every endpoint returns data in a different format. Don't be that API.

---

## Request design

### Request structure

A typical HTTP request has:

```http
POST /users HTTP/1.1                    ← Method + Path + Protocol
Host: api.example.com                   ← Headers
Content-Type: application/json          ← Headers
Authorization: Bearer eyJhbGc...        ← Headers
Accept: application/json                ← Headers
                                        ← Empty line
{                                       ← Body
  "name": "John Doe",
  "email": "john@example.com"
}
```

### Headers

Headers provide metadata about the request.

**Common request headers:**

| Header | Purpose | Example |
|--------|---------|---------|
| `Content-Type` | Format of request body | `application/json` |
| `Accept` | Desired response format | `application/json` |
| `Authorization` | Authentication credentials | `Bearer eyJhbGc...` |
| `Accept-Language` | Preferred language | `en-US, en;q=0.9` |
| `Accept-Encoding` | Supported compression | `gzip, deflate` |
| `If-None-Match` | Conditional request (caching) | `"abc123"` |
| `If-Modified-Since` | Conditional request (caching) | `Mon, 15 Jan 2024 10:30:00 GMT` |
| `Idempotency-Key` | Prevent duplicate operations | `order-abc-123` |
| `X-Request-ID` | Request tracing | `req-123-456-789` |

**Custom headers:**

Use `X-` prefix for custom headers (though this convention is deprecated, it's still common):

```http
X-Tenant-ID: acme-corp
X-API-Version: 2024-01-15
X-Request-Source: mobile-app
```

### Request body formats

**JSON (most common)**
```http
POST /users HTTP/1.1
Content-Type: application/json

{
  "name": "John Doe",
  "email": "john@example.com",
  "preferences": {
    "newsletter": true,
    "notifications": ["email", "push"]
  }
}
```

**Form data (for file uploads)**
```http
POST /users/123/avatar HTTP/1.1
Content-Type: multipart/form-data; boundary=----WebKitFormBoundary

------WebKitFormBoundary
Content-Disposition: form-data; name="file"; filename="avatar.jpg"
Content-Type: image/jpeg

[binary data]
------WebKitFormBoundary--
```

**URL-encoded (legacy, OAuth)**
```http
POST /oauth/token HTTP/1.1
Content-Type: application/x-www-form-urlencoded

grant_type=password&username=john&password=secret
```

### Request body best practices

**1. Use consistent field naming**

Pick a convention and stick to it:

```json
// snake_case (common in Ruby, Python APIs)
{
  "first_name": "John",
  "last_name": "Doe",
  "created_at": "2024-01-15T10:30:00Z"
}

// camelCase (common in JavaScript APIs)
{
  "firstName": "John",
  "lastName": "Doe",
  "createdAt": "2024-01-15T10:30:00Z"
}
```

**2. Use appropriate data types**

```json
// Good
{
  "age": 25,                           // number, not "25"
  "is_active": true,                   // boolean, not "true"
  "price": 29.99,                      // number, not "29.99"
  "created_at": "2024-01-15T10:30:00Z" // ISO 8601 string
}

// Bad
{
  "age": "25",
  "is_active": "true",
  "price": "29.99",
  "created_at": "January 15, 2024"
}
```

**3. Use ISO 8601 for dates**

```json
{
  "created_at": "2024-01-15T10:30:00Z",        // UTC
  "updated_at": "2024-01-15T10:30:00+05:30",   // With timezone
  "date_of_birth": "1990-05-20"                // Date only
}
```

**4. Nest related data appropriately**

```json
// Good: Logical grouping
{
  "name": "John Doe",
  "address": {
    "street": "123 Main St",
    "city": "New York",
    "country": "US"
  },
  "preferences": {
    "newsletter": true,
    "language": "en"
  }
}

// Bad: Flat structure with prefixes
{
  "name": "John Doe",
  "address_street": "123 Main St",
  "address_city": "New York",
  "address_country": "US",
  "preferences_newsletter": true,
  "preferences_language": "en"
}
```

**5. Handle null vs. absent fields**

```json
// Field is absent: Use default or don't change
{
  "name": "John"
  // email not included - keep existing value
}

// Field is null: Explicitly clear the value
{
  "name": "John",
  "email": null  // Clear the email
}

// Field is empty string: Set to empty
{
  "name": "John",
  "email": ""  // Set email to empty string
}
```

Document your convention clearly!

---

## Response design

### Response structure

A typical HTTP response has:

```http
HTTP/1.1 200 OK                         ← Status line
Content-Type: application/json          ← Headers
X-Request-ID: req-123-456-789           ← Headers
Cache-Control: max-age=3600             ← Headers
                                        ← Empty line
{                                       ← Body
  "id": "123",
  "name": "John Doe",
  "email": "john@example.com"
}
```

### Response headers

**Common response headers:**

| Header | Purpose | Example |
|--------|---------|---------|
| `Content-Type` | Format of response body | `application/json` |
| `Content-Length` | Size of response body | `256` |
| `Cache-Control` | Caching directives | `max-age=3600` |
| `ETag` | Resource version for caching | `"abc123"` |
| `Last-Modified` | When resource was last changed | `Mon, 15 Jan 2024 10:30:00 GMT` |
| `Location` | URL of created resource | `/users/123` |
| `X-Request-ID` | Request tracing | `req-123-456-789` |
| `X-RateLimit-Limit` | Rate limit ceiling | `1000` |
| `X-RateLimit-Remaining` | Remaining requests | `999` |
| `X-RateLimit-Reset` | When limit resets | `1705312200` |

### Response body structure

**Consistent envelope (optional but recommended)**

```json
{
  "data": {
    "id": "123",
    "name": "John Doe"
  },
  "meta": {
    "request_id": "req-123-456-789",
    "timestamp": "2024-01-15T10:30:00Z"
  }
}
```

**For collections:**

```json
{
  "data": [
    {"id": "123", "name": "John"},
    {"id": "456", "name": "Jane"}
  ],
  "pagination": {
    "total": 100,
    "page": 1,
    "limit": 20,
    "has_more": true
  },
  "meta": {
    "request_id": "req-123-456-789"
  }
}
```

**For errors:**

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "The request body is invalid",
    "details": [
      {
        "field": "email",
        "message": "Must be a valid email address"
      }
    ]
  },
  "meta": {
    "request_id": "req-123-456-789"
  }
}
```

### Response design patterns

**Pattern 1: Direct response (simple)**

```json
// GET /users/123
{
  "id": "123",
  "name": "John Doe",
  "email": "john@example.com"
}
```

Pros: Simple, less nesting
Cons: Hard to add metadata later

**Pattern 2: Envelope response**

```json
// GET /users/123
{
  "data": {
    "id": "123",
    "name": "John Doe",
    "email": "john@example.com"
  }
}
```

Pros: Consistent structure, easy to add metadata
Cons: Extra nesting

**Pattern 3: Full envelope with metadata**

```json
// GET /users/123
{
  "data": {
    "id": "123",
    "name": "John Doe",
    "email": "john@example.com"
  },
  "meta": {
    "request_id": "req-123",
    "timestamp": "2024-01-15T10:30:00Z",
    "version": "v1"
  },
  "links": {
    "self": "/users/123",
    "orders": "/users/123/orders"
  }
}
```

Pros: Rich metadata, HATEOAS support
Cons: Verbose

**My recommendation:** Use Pattern 2 (envelope) for consistency. It makes it easier to add pagination, metadata, and error handling without breaking changes.

---

## Content negotiation

Content negotiation allows clients to request different formats.

### Request format

The `Accept` header specifies what format the client wants:

```http
GET /users/123 HTTP/1.1
Accept: application/json

GET /users/123 HTTP/1.1
Accept: application/xml

GET /users/123 HTTP/1.1
Accept: text/csv
```

### Response format

The `Content-Type` header specifies what format the server is returning:

```http
HTTP/1.1 200 OK
Content-Type: application/json

{"id": "123", "name": "John"}
```

```http
HTTP/1.1 200 OK
Content-Type: application/xml

<user><id>123</id><name>John</name></user>
```

### Quality values

Clients can specify preferences with quality values:

```http
Accept: application/json, application/xml;q=0.9, */*;q=0.8
```

This means: prefer JSON, then XML, then anything else.

### Handling unsupported formats

If the server can't provide the requested format:

```http
GET /users/123 HTTP/1.1
Accept: application/pdf

HTTP/1.1 406 Not Acceptable
Content-Type: application/json

{
  "error": {
    "code": "NOT_ACCEPTABLE",
    "message": "Requested format not supported",
    "supported_formats": ["application/json", "application/xml"]
  }
}
```

---

## Field selection and expansion

### Field selection (sparse fieldsets)

Let clients request only the fields they need:

```http
GET /users/123?fields=id,name,email

{
  "id": "123",
  "name": "John Doe",
  "email": "john@example.com"
}
```

Without field selection:

```http
GET /users/123

{
  "id": "123",
  "name": "John Doe",
  "email": "john@example.com",
  "phone": "+1234567890",
  "address": {...},
  "preferences": {...},
  "created_at": "...",
  "updated_at": "...",
  // ... 20 more fields
}
```

### Field expansion (embedding related resources)

Let clients request related resources in a single call:

```http
GET /orders/123?expand=customer,items

{
  "id": "123",
  "status": "pending",
  "customer": {                    // Expanded
    "id": "456",
    "name": "John Doe",
    "email": "john@example.com"
  },
  "items": [                       // Expanded
    {
      "id": "789",
      "product_name": "Widget",
      "quantity": 2,
      "price": 29.99
    }
  ],
  "total": 59.98
}
```

Without expansion:

```http
GET /orders/123

{
  "id": "123",
  "status": "pending",
  "customer_id": "456",           // Just the ID
  "item_ids": ["789", "790"],     // Just the IDs
  "total": 59.98
}
```

### Nested expansion

Some APIs support nested expansion:

```http
GET /orders/123?expand=customer.addresses,items.product

{
  "id": "123",
  "customer": {
    "id": "456",
    "name": "John Doe",
    "addresses": [                 // Nested expansion
      {"id": "1", "city": "New York"}
    ]
  },
  "items": [
    {
      "id": "789",
      "quantity": 2,
      "product": {                 // Nested expansion
        "id": "abc",
        "name": "Widget"
      }
    }
  ]
}
```

---

## Handling relationships

### Option 1: Nested resources

Include related resources directly:

```json
{
  "id": "123",
  "name": "John Doe",
  "orders": [
    {"id": "456", "total": 99.99},
    {"id": "789", "total": 149.99}
  ]
}
```

Pros: One request gets everything
Cons: Response can be huge, circular references are tricky

### Option 2: Resource IDs only

Include only IDs, let client fetch if needed:

```json
{
  "id": "123",
  "name": "John Doe",
  "order_ids": ["456", "789"]
}
```

Pros: Small response, no circular references
Cons: Multiple requests needed

### Option 3: Links (HATEOAS)

Include URLs to related resources:

```json
{
  "id": "123",
  "name": "John Doe",
  "_links": {
    "orders": "/users/123/orders",
    "addresses": "/users/123/addresses"
  }
}
```

Pros: Discoverable, flexible
Cons: Multiple requests needed

### Option 4: Hybrid (recommended)

Include IDs and links, with optional expansion:

```json
{
  "id": "123",
  "name": "John Doe",
  "order_ids": ["456", "789"],
  "_links": {
    "orders": "/users/123/orders"
  }
}

// With expansion: GET /users/123?expand=orders
{
  "id": "123",
  "name": "John Doe",
  "orders": [
    {"id": "456", "total": 99.99},
    {"id": "789", "total": 149.99}
  ],
  "_links": {
    "orders": "/users/123/orders"
  }
}
```

---

## Handling large payloads

### Compression

Use gzip compression for large responses:

```http
GET /users HTTP/1.1
Accept-Encoding: gzip

HTTP/1.1 200 OK
Content-Encoding: gzip
Content-Length: 1234

[compressed data]
```

### Streaming

For very large responses, use streaming:

```http
GET /reports/large-export HTTP/1.1

HTTP/1.1 200 OK
Transfer-Encoding: chunked
Content-Type: application/json

{"users": [
{"id": "1", "name": "John"},
{"id": "2", "name": "Jane"},
...
]}
```

### Pagination

For collections, always paginate:

```http
GET /users?page=1&limit=100

{
  "data": [...],
  "pagination": {
    "total": 10000,
    "page": 1,
    "limit": 100,
    "pages": 100
  }
}
```

---

## Request validation

### Validate early, fail fast

```json
// Request
POST /users
{
  "name": "",
  "email": "not-an-email",
  "age": -5
}

// Response
HTTP/1.1 400 Bad Request
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Request validation failed",
    "details": [
      {"field": "name", "message": "Name is required"},
      {"field": "email", "message": "Must be a valid email"},
      {"field": "age", "message": "Must be a positive number"}
    ]
  }
}
```

### Common validations

| Field type | Validations |
|------------|-------------|
| String | Required, min/max length, pattern (regex), enum |
| Number | Required, min/max value, integer vs. decimal |
| Email | Required, valid format |
| URL | Required, valid format, allowed protocols |
| Date | Required, valid format, min/max date |
| Array | Required, min/max items, item validation |
| Object | Required fields, nested validation |

### Schema validation

Use JSON Schema or similar for validation:

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "required": ["name", "email"],
  "properties": {
    "name": {
      "type": "string",
      "minLength": 1,
      "maxLength": 100
    },
    "email": {
      "type": "string",
      "format": "email"
    },
    "age": {
      "type": "integer",
      "minimum": 0,
      "maximum": 150
    }
  }
}
```

---

## Idempotency and request IDs

### Idempotency keys

For non-idempotent operations (POST), use idempotency keys:

```http
POST /orders HTTP/1.1
Idempotency-Key: order-abc-123-xyz
Content-Type: application/json

{
  "product_id": "456",
  "quantity": 1
}
```

Server behavior:
1. First request: Process and store result with key
2. Subsequent requests with same key: Return stored result
3. Key expires after some time (e.g., 24 hours)

### Request IDs

Include a request ID for tracing:

```http
POST /orders HTTP/1.1
X-Request-ID: req-123-456-789

HTTP/1.1 201 Created
X-Request-ID: req-123-456-789

{
  "id": "order-789",
  "meta": {
    "request_id": "req-123-456-789"
  }
}
```

If the client doesn't provide one, the server should generate it.

---

## Common patterns

### Bulk operations

**Request:**
```json
POST /users/bulk
{
  "operations": [
    {"action": "create", "data": {"name": "John", "email": "john@example.com"}},
    {"action": "update", "id": "123", "data": {"name": "Jane"}},
    {"action": "delete", "id": "456"}
  ]
}
```

**Response:**
```json
{
  "results": [
    {"status": "success", "id": "789", "action": "create"},
    {"status": "success", "id": "123", "action": "update"},
    {"status": "error", "id": "456", "action": "delete", "error": "Not found"}
  ],
  "summary": {
    "total": 3,
    "succeeded": 2,
    "failed": 1
  }
}
```

### Async operations

**Request:**
```http
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
  "status_url": "/jobs/abc123",
  "estimated_completion": "2024-01-15T11:00:00Z"
}
```

**Polling:**
```http
GET /jobs/abc123

{
  "job_id": "abc123",
  "status": "completed",
  "result_url": "/reports/xyz789",
  "completed_at": "2024-01-15T10:45:00Z"
}
```

### Webhooks

For async notifications:

```json
POST /webhooks/your-endpoint
{
  "event": "order.completed",
  "timestamp": "2024-01-15T10:30:00Z",
  "data": {
    "order_id": "123",
    "total": 99.99
  }
}
```

---

## Common mistakes

### 1. Inconsistent field naming

```json
// Bad: Mixed conventions
{
  "firstName": "John",
  "last_name": "Doe",
  "EmailAddress": "john@example.com"
}

// Good: Consistent convention
{
  "first_name": "John",
  "last_name": "Doe",
  "email_address": "john@example.com"
}
```

### 2. Returning different structures for same resource

```json
// GET /users/123
{"id": "123", "name": "John", "email": "john@example.com"}

// GET /users (list)
{"users": [{"user_id": "123", "full_name": "John"}]}  // Different fields!
```

### 3. Not including resource ID in response

```json
// Bad
POST /users → {"name": "John"}  // Where's the ID?

// Good
POST /users → {"id": "123", "name": "John"}
```

### 4. Returning arrays at root level

```json
// Bad: Can't add metadata later
[{"id": "1"}, {"id": "2"}]

// Good: Wrapped in object
{"data": [{"id": "1"}, {"id": "2"}]}
```

### 5. Inconsistent null handling

```json
// Inconsistent
{"name": "John", "email": null, "phone": ""}  // null vs empty string?

// Document your convention!
```

---

## Key takeaways

1. **Be consistent.** Same field names, same structure, same conventions everywhere.

2. **Use envelopes.** Wrap responses in `{"data": ...}` for flexibility.

3. **Validate early.** Return all validation errors at once.

4. **Support field selection.** Let clients request only what they need.

5. **Use ISO 8601 for dates.** Always.

6. **Include request IDs.** Essential for debugging.

7. **Document null handling.** Absent vs. null vs. empty string.

8. **Compress large responses.** Use gzip.

---

[Next: Status Codes & Error Handling →](./06-status-codes-and-errors.md)