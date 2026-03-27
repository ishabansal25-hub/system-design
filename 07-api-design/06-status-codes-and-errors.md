# Status Codes & Error Handling

[← Back to API Design](./README.md) | [← Previous: Request & Response Design](./05-request-response-design.md)

---

HTTP status codes are how your API communicates what happened. Use them correctly, and developers can handle responses without even reading the body. Use them incorrectly, and you'll have confused developers and broken integrations.

I've seen APIs that return `200 OK` for everything, including errors. The body would say `{"success": false, "error": "User not found"}`. Don't do this. It breaks HTTP semantics, confuses caching layers, and makes error handling a nightmare.

---

## Status code categories

HTTP status codes are grouped into five categories:

| Range | Category | Meaning |
|-------|----------|---------|
| 1xx | Informational | Request received, continuing process |
| 2xx | Success | Request successfully received, understood, and accepted |
| 3xx | Redirection | Further action needed to complete the request |
| 4xx | Client Error | Request contains bad syntax or cannot be fulfilled |
| 5xx | Server Error | Server failed to fulfill a valid request |

**The simple rule:** 
- 2xx = "It worked"
- 4xx = "You (the client) messed up"
- 5xx = "We (the server) messed up"

---

## Success codes (2xx)

### 200 OK

The request succeeded. The response body contains the result.

```http
GET /users/123 HTTP/1.1

HTTP/1.1 200 OK
Content-Type: application/json

{
  "id": "123",
  "name": "John Doe"
}
```

**Use for:** GET requests, PUT/PATCH updates that return the updated resource.

### 201 Created

A new resource was created successfully.

```http
POST /users HTTP/1.1
Content-Type: application/json

{"name": "John Doe", "email": "john@example.com"}

HTTP/1.1 201 Created
Location: /users/123
Content-Type: application/json

{
  "id": "123",
  "name": "John Doe",
  "email": "john@example.com"
}
```

**Use for:** POST requests that create resources. Include the `Location` header pointing to the new resource.

### 202 Accepted

The request has been accepted for processing, but processing is not complete.

```http
POST /reports/generate HTTP/1.1
Content-Type: application/json

{"type": "sales", "year": 2024}

HTTP/1.1 202 Accepted
Location: /jobs/abc123
Content-Type: application/json

{
  "job_id": "abc123",
  "status": "pending",
  "status_url": "/jobs/abc123"
}
```

**Use for:** Async operations, batch jobs, anything that takes time.

### 204 No Content

The request succeeded, but there's no content to return.

```http
DELETE /users/123 HTTP/1.1

HTTP/1.1 204 No Content
```

**Use for:** DELETE requests, PUT/PATCH when you don't need to return the updated resource.

### 206 Partial Content

The server is returning part of the resource (for range requests).

```http
GET /files/large-video.mp4 HTTP/1.1
Range: bytes=0-1023

HTTP/1.1 206 Partial Content
Content-Range: bytes 0-1023/1048576
Content-Length: 1024

[first 1024 bytes of video]
```

**Use for:** Large file downloads, video streaming, resumable downloads.

---

## Redirection codes (3xx)

### 301 Moved Permanently

The resource has permanently moved to a new URL.

```http
GET /old-endpoint HTTP/1.1

HTTP/1.1 301 Moved Permanently
Location: /new-endpoint
```

**Use for:** Permanent URL changes. Clients should update their bookmarks.

### 302 Found (Temporary Redirect)

The resource is temporarily at a different URL.

```http
GET /users/123/avatar HTTP/1.1

HTTP/1.1 302 Found
Location: https://cdn.example.com/avatars/123.jpg
```

**Use for:** Temporary redirects, like redirecting to a CDN.

### 304 Not Modified

The resource hasn't changed since the client's cached version.

```http
GET /users/123 HTTP/1.1
If-None-Match: "abc123"

HTTP/1.1 304 Not Modified
ETag: "abc123"
```

**Use for:** Caching. Client can use its cached copy.

### 307 Temporary Redirect

Like 302, but the client must use the same HTTP method.

```http
POST /old-endpoint HTTP/1.1

HTTP/1.1 307 Temporary Redirect
Location: /new-endpoint
```

**Use for:** When you need to preserve the HTTP method during redirect.

### 308 Permanent Redirect

Like 301, but the client must use the same HTTP method.

```http
POST /old-endpoint HTTP/1.1

HTTP/1.1 308 Permanent Redirect
Location: /new-endpoint
```

**Use for:** Permanent redirects that must preserve the HTTP method.

---

## Client error codes (4xx)

### 400 Bad Request

The request is malformed or invalid.

```http
POST /users HTTP/1.1
Content-Type: application/json

{"name": 123}  // name should be a string

HTTP/1.1 400 Bad Request
Content-Type: application/json

{
  "error": {
    "code": "INVALID_REQUEST",
    "message": "The request body is invalid",
    "details": [
      {
        "field": "name",
        "message": "Expected string, got number"
      }
    ]
  }
}
```

**Use for:** Malformed JSON, invalid data types, missing required fields.

### 401 Unauthorized

Authentication is required but was not provided or is invalid.

```http
GET /users/123 HTTP/1.1

HTTP/1.1 401 Unauthorized
WWW-Authenticate: Bearer realm="api"
Content-Type: application/json

{
  "error": {
    "code": "UNAUTHORIZED",
    "message": "Authentication required"
  }
}
```

**Use for:** Missing or invalid authentication credentials. Include `WWW-Authenticate` header.

**Note:** Despite the name, this is about *authentication* (who you are), not *authorization* (what you can do).

### 403 Forbidden

The client is authenticated but doesn't have permission.

```http
DELETE /users/456 HTTP/1.1
Authorization: Bearer eyJhbGc...

HTTP/1.1 403 Forbidden
Content-Type: application/json

{
  "error": {
    "code": "FORBIDDEN",
    "message": "You don't have permission to delete this user"
  }
}
```

**Use for:** Authenticated user trying to access something they're not allowed to.

### 404 Not Found

The requested resource doesn't exist.

```http
GET /users/999 HTTP/1.1

HTTP/1.1 404 Not Found
Content-Type: application/json

{
  "error": {
    "code": "NOT_FOUND",
    "message": "User not found"
  }
}
```

**Use for:** Resource doesn't exist. Also use when you don't want to reveal that a resource exists (for security).

### 405 Method Not Allowed

The HTTP method is not supported for this resource.

```http
DELETE /users HTTP/1.1

HTTP/1.1 405 Method Not Allowed
Allow: GET, POST
Content-Type: application/json

{
  "error": {
    "code": "METHOD_NOT_ALLOWED",
    "message": "DELETE is not allowed on /users"
  }
}
```

**Use for:** Wrong HTTP method. Include `Allow` header listing valid methods.

### 406 Not Acceptable

The server can't produce a response matching the `Accept` header.

```http
GET /users/123 HTTP/1.1
Accept: application/pdf

HTTP/1.1 406 Not Acceptable
Content-Type: application/json

{
  "error": {
    "code": "NOT_ACCEPTABLE",
    "message": "Cannot produce application/pdf",
    "supported_formats": ["application/json", "application/xml"]
  }
}
```

**Use for:** Client requested a format the server doesn't support.

### 409 Conflict

The request conflicts with the current state of the resource.

```http
POST /users HTTP/1.1
Content-Type: application/json

{"email": "john@example.com"}

HTTP/1.1 409 Conflict
Content-Type: application/json

{
  "error": {
    "code": "CONFLICT",
    "message": "A user with this email already exists",
    "existing_resource": "/users/123"
  }
}
```

**Use for:** Duplicate resources, version conflicts, state conflicts.

### 410 Gone

The resource existed but has been permanently deleted.

```http
GET /users/123 HTTP/1.1

HTTP/1.1 410 Gone
Content-Type: application/json

{
  "error": {
    "code": "GONE",
    "message": "This user has been deleted",
    "deleted_at": "2024-01-15T10:30:00Z"
  }
}
```

**Use for:** Permanently deleted resources. Different from 404 - this tells the client the resource used to exist.

### 415 Unsupported Media Type

The server doesn't support the request's content type.

```http
POST /users HTTP/1.1
Content-Type: application/xml

<user><name>John</name></user>

HTTP/1.1 415 Unsupported Media Type
Content-Type: application/json

{
  "error": {
    "code": "UNSUPPORTED_MEDIA_TYPE",
    "message": "application/xml is not supported",
    "supported_types": ["application/json"]
  }
}
```

**Use for:** Client sent data in a format the server doesn't accept.

### 422 Unprocessable Entity

The request is well-formed but contains semantic errors.

```http
POST /users HTTP/1.1
Content-Type: application/json

{
  "name": "John",
  "email": "not-an-email",
  "age": -5
}

HTTP/1.1 422 Unprocessable Entity
Content-Type: application/json

{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Validation failed",
    "details": [
      {"field": "email", "message": "Must be a valid email address"},
      {"field": "age", "message": "Must be a positive number"}
    ]
  }
}
```

**Use for:** Validation errors. The JSON is valid, but the data doesn't make sense.

**400 vs. 422:** Use 400 for malformed requests (bad JSON), 422 for valid requests with invalid data.

### 429 Too Many Requests

The client has sent too many requests (rate limiting).

```http
GET /users HTTP/1.1

HTTP/1.1 429 Too Many Requests
Retry-After: 60
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1705312200
Content-Type: application/json

{
  "error": {
    "code": "RATE_LIMITED",
    "message": "Too many requests",
    "retry_after": 60
  }
}
```

**Use for:** Rate limiting. Include `Retry-After` header.

---

## Server error codes (5xx)

### 500 Internal Server Error

Something went wrong on the server.

```http
GET /users/123 HTTP/1.1

HTTP/1.1 500 Internal Server Error
Content-Type: application/json

{
  "error": {
    "code": "INTERNAL_ERROR",
    "message": "An unexpected error occurred",
    "request_id": "req-123-456-789"
  }
}
```

**Use for:** Unexpected errors. Don't expose internal details to clients.

### 502 Bad Gateway

The server got an invalid response from an upstream server.

```http
GET /users/123 HTTP/1.1

HTTP/1.1 502 Bad Gateway
Content-Type: application/json

{
  "error": {
    "code": "BAD_GATEWAY",
    "message": "Unable to reach upstream service"
  }
}
```

**Use for:** When your API depends on another service that's returning errors.

### 503 Service Unavailable

The server is temporarily unavailable (maintenance, overload).

```http
GET /users/123 HTTP/1.1

HTTP/1.1 503 Service Unavailable
Retry-After: 300
Content-Type: application/json

{
  "error": {
    "code": "SERVICE_UNAVAILABLE",
    "message": "Service is temporarily unavailable",
    "retry_after": 300
  }
}
```

**Use for:** Planned maintenance, temporary overload. Include `Retry-After` header.

### 504 Gateway Timeout

The server didn't get a timely response from an upstream server.

```http
GET /reports/complex-query HTTP/1.1

HTTP/1.1 504 Gateway Timeout
Content-Type: application/json

{
  "error": {
    "code": "GATEWAY_TIMEOUT",
    "message": "Request timed out"
  }
}
```

**Use for:** Upstream service timeouts.

---

## Error response design

### Consistent error structure

Every error should have the same structure:

```json
{
  "error": {
    "code": "ERROR_CODE",           // Machine-readable code
    "message": "Human-readable message",
    "details": [...],               // Optional: additional info
    "request_id": "req-123"         // For debugging
  }
}
```

### Error codes

Use consistent, descriptive error codes:

| Code | Meaning |
|------|---------|
| `VALIDATION_ERROR` | Request validation failed |
| `AUTHENTICATION_REQUIRED` | No authentication provided |
| `INVALID_TOKEN` | Token is invalid or expired |
| `FORBIDDEN` | Not authorized for this action |
| `NOT_FOUND` | Resource doesn't exist |
| `CONFLICT` | Resource conflict (duplicate, etc.) |
| `RATE_LIMITED` | Too many requests |
| `INTERNAL_ERROR` | Server error |

### Validation error details

For validation errors, include field-level details:

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Request validation failed",
    "details": [
      {
        "field": "email",
        "code": "INVALID_FORMAT",
        "message": "Must be a valid email address"
      },
      {
        "field": "password",
        "code": "TOO_SHORT",
        "message": "Must be at least 8 characters",
        "min_length": 8
      },
      {
        "field": "items[0].quantity",
        "code": "OUT_OF_RANGE",
        "message": "Must be between 1 and 100",
        "min": 1,
        "max": 100
      }
    ]
  }
}
```

### Localized error messages

Support multiple languages:

```http
GET /users/123 HTTP/1.1
Accept-Language: es

HTTP/1.1 404 Not Found
Content-Type: application/json

{
  "error": {
    "code": "NOT_FOUND",
    "message": "Usuario no encontrado"
  }
}
```

Or return both:

```json
{
  "error": {
    "code": "NOT_FOUND",
    "message": "User not found",
    "localized_message": "Usuario no encontrado"
  }
}
```

### Error documentation links

Include links to documentation:

```json
{
  "error": {
    "code": "RATE_LIMITED",
    "message": "Too many requests",
    "documentation_url": "https://api.example.com/docs/rate-limiting"
  }
}
```

---

## Error handling best practices

### 1. Be specific

```json
// Bad
{
  "error": "Something went wrong"
}

// Good
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "The email field must be a valid email address",
    "field": "email"
  }
}
```

### 2. Don't expose internal details

```json
// Bad (exposes internal info)
{
  "error": {
    "message": "SQLException: Connection refused to mysql://prod-db:3306",
    "stack_trace": "at com.example.UserRepository.findById..."
  }
}

// Good
{
  "error": {
    "code": "INTERNAL_ERROR",
    "message": "An unexpected error occurred",
    "request_id": "req-123-456-789"
  }
}
```

Log the details server-side, but don't send them to clients.

### 3. Return all validation errors at once

```json
// Bad (one error at a time)
{
  "error": {
    "field": "email",
    "message": "Invalid email"
  }
}
// Client fixes email, submits again...
{
  "error": {
    "field": "password",
    "message": "Too short"
  }
}

// Good (all errors at once)
{
  "error": {
    "code": "VALIDATION_ERROR",
    "details": [
      {"field": "email", "message": "Invalid email"},
      {"field": "password", "message": "Too short"},
      {"field": "age", "message": "Must be positive"}
    ]
  }
}
```

### 4. Use appropriate status codes

```
// Bad
HTTP/1.1 200 OK
{"success": false, "error": "User not found"}

// Good
HTTP/1.1 404 Not Found
{"error": {"code": "NOT_FOUND", "message": "User not found"}}
```

### 5. Include request ID for debugging

```json
{
  "error": {
    "code": "INTERNAL_ERROR",
    "message": "An unexpected error occurred",
    "request_id": "req-123-456-789"
  }
}
```

Clients can include this in support requests. You can use it to find logs.

### 6. Provide actionable guidance

```json
// Bad
{
  "error": {
    "message": "Invalid request"
  }
}

// Good
{
  "error": {
    "code": "INVALID_API_KEY",
    "message": "The API key is invalid or expired",
    "hint": "Check that your API key is correct and hasn't expired",
    "documentation_url": "https://api.example.com/docs/authentication"
  }
}
```

---

## Status code decision tree

```
Did the request succeed?
├── Yes
│   ├── Created something? → 201 Created
│   ├── Async processing? → 202 Accepted
│   ├── No content to return? → 204 No Content
│   └── Otherwise → 200 OK
│
└── No
    ├── Client's fault?
    │   ├── Bad syntax/format? → 400 Bad Request
    │   ├── Not authenticated? → 401 Unauthorized
    │   ├── Not authorized? → 403 Forbidden
    │   ├── Resource not found? → 404 Not Found
    │   ├── Wrong HTTP method? → 405 Method Not Allowed
    │   ├── Conflict? → 409 Conflict
    │   ├── Validation error? → 422 Unprocessable Entity
    │   └── Rate limited? → 429 Too Many Requests
    │
    └── Server's fault?
        ├── Unexpected error? → 500 Internal Server Error
        ├── Upstream error? → 502 Bad Gateway
        ├── Temporarily unavailable? → 503 Service Unavailable
        └── Upstream timeout? → 504 Gateway Timeout
```

---

## Common mistakes

### 1. Using 200 for errors

```
// Bad
HTTP/1.1 200 OK
{"success": false, "error": "Not found"}

// Good
HTTP/1.1 404 Not Found
{"error": {"code": "NOT_FOUND", "message": "Not found"}}
```

### 2. Using 500 for client errors

```
// Bad
HTTP/1.1 500 Internal Server Error
{"error": "Invalid email format"}

// Good
HTTP/1.1 422 Unprocessable Entity
{"error": {"code": "VALIDATION_ERROR", "message": "Invalid email format"}}
```

### 3. Inconsistent error formats

```
// Bad: Different formats for different errors
{"error": "Not found"}
{"message": "Validation failed", "errors": [...]}
{"status": "error", "reason": "Unauthorized"}

// Good: Same format everywhere
{"error": {"code": "...", "message": "..."}}
```

### 4. Missing error details

```
// Bad
{"error": "Bad request"}

// Good
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Request validation failed",
    "details": [
      {"field": "email", "message": "Must be a valid email"}
    ]
  }
}
```

### 5. Exposing sensitive information

```
// Bad
{
  "error": {
    "message": "User john@example.com not found in database users_prod"
  }
}

// Good
{
  "error": {
    "code": "NOT_FOUND",
    "message": "User not found"
  }
}
```

---

## Key takeaways

1. **Use the right status code.** 2xx for success, 4xx for client errors, 5xx for server errors.

2. **Be consistent.** Same error format everywhere.

3. **Be specific.** Tell developers exactly what went wrong.

4. **Be helpful.** Include hints, documentation links, and actionable guidance.

5. **Be secure.** Don't expose internal details in error messages.

6. **Include request IDs.** Essential for debugging.

7. **Return all validation errors at once.** Don't make developers play whack-a-mole.

---

[Next: Versioning Strategies →](./07-versioning-strategies.md)