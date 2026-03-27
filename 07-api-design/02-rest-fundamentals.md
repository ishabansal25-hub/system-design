# REST Fundamentals

[← Back to API Design](./README.md) | [← Previous: What is API Design?](./01-what-is-api-design.md)

---

REST (Representational State Transfer) isn't a protocol or a standard - it's an architectural style. Roy Fielding defined it in his 2000 PhD dissertation, and it's become the dominant approach for web APIs.

But here's the thing: most "REST APIs" aren't actually RESTful. They're just HTTP APIs that use JSON. And honestly? That's fine for most use cases. But understanding true REST helps you make better design decisions.

---

## The six REST constraints

REST is defined by six architectural constraints. An API that follows all of them is "RESTful." Most APIs follow some of them.

### 1. Client-Server

The client and server are separate. They can evolve independently as long as the interface between them stays the same.

```
Client (mobile app, web browser, another service)
    ↓
    HTTP Request
    ↓
Server (your API)
    ↓
    HTTP Response
    ↓
Client
```

**Why it matters:** You can update your server without updating every client. You can have multiple clients (web, mobile, CLI) using the same API.

### 2. Stateless

Each request contains all the information needed to process it. The server doesn't store client state between requests.

```
Bad (stateful):
Request 1: POST /login {"user": "john", "pass": "secret"}
           Server stores: session_id = abc123, user = john
Request 2: GET /my-orders
           Server looks up session_id, finds user = john

Good (stateless):
Request 1: POST /login {"user": "john", "pass": "secret"}
           Response: {"token": "eyJhbGc..."}
Request 2: GET /my-orders
           Header: Authorization: Bearer eyJhbGc...
           Server decodes token, finds user = john
```

**Why it matters:**
- **Scalability**: Any server can handle any request. No need for sticky sessions.
- **Reliability**: If a server dies, another can take over. No session state is lost.
- **Simplicity**: Servers don't need to manage session storage.

**The tradeoff:** Every request must include authentication info, which adds overhead.

### 3. Cacheable

Responses should indicate whether they can be cached and for how long.

```http
HTTP/1.1 200 OK
Cache-Control: max-age=3600
ETag: "abc123"

{
  "id": "123",
  "name": "Product Name",
  "price": 29.99
}
```

**Cache-Control directives:**
| Directive | Meaning |
|-----------|---------|
| `max-age=3600` | Cache for 1 hour |
| `no-cache` | Must revalidate before using cached version |
| `no-store` | Don't cache at all |
| `private` | Only browser can cache, not CDN |
| `public` | Anyone can cache |

**Why it matters:** Caching reduces server load and improves response times. A well-cached API can handle 10x the traffic.

### 4. Uniform Interface

This is the big one. REST APIs should have a consistent, predictable interface. It has four sub-constraints:

#### a) Resource identification

Resources are identified by URLs. Each resource has a unique URL.

```
/users/123          → User with ID 123
/orders/456         → Order with ID 456
/users/123/orders   → Orders belonging to user 123
```

#### b) Resource manipulation through representations

When you GET a resource, you get a representation of it (usually JSON). When you want to modify it, you send a representation of the desired state.

```
GET /users/123
Response:
{
  "id": "123",
  "name": "John",
  "email": "john@example.com"
}

PUT /users/123
Request:
{
  "id": "123",
  "name": "John Doe",        ← Changed
  "email": "john@example.com"
}
```

#### c) Self-descriptive messages

Each message contains enough information to understand how to process it.

```http
POST /users HTTP/1.1
Host: api.example.com
Content-Type: application/json    ← Tells server how to parse body
Accept: application/json          ← Tells server what format to return
Authorization: Bearer eyJhbGc...  ← Authentication

{
  "name": "John",
  "email": "john@example.com"
}
```

#### d) HATEOAS (Hypermedia as the Engine of Application State)

Responses should include links to related resources and available actions.

```json
{
  "id": "123",
  "name": "John",
  "email": "john@example.com",
  "_links": {
    "self": {"href": "/users/123"},
    "orders": {"href": "/users/123/orders"},
    "update": {"href": "/users/123", "method": "PUT"},
    "delete": {"href": "/users/123", "method": "DELETE"}
  }
}
```

**Why HATEOAS matters:** Clients don't need to hardcode URLs. They can discover available actions from the API itself. In practice, most APIs skip this because it adds complexity and most clients don't use it.

### 5. Layered System

The client doesn't know (or care) if it's talking directly to the server or through intermediaries like load balancers, caches, or gateways.

```
Client → CDN → Load Balancer → API Gateway → Server
         ↑         ↑              ↑
    (caching)  (routing)    (auth, rate limiting)
```

**Why it matters:** You can add layers for caching, security, and load balancing without changing the client.

### 6. Code on Demand (Optional)

Servers can send executable code to clients (like JavaScript). This is the only optional constraint and is rarely used in APIs.

---

## Resources: The heart of REST

In REST, everything is a resource. A resource is any concept that can be named and addressed:

- A user
- An order
- A product
- A collection of products
- A search result
- A user's shopping cart

### Resource naming conventions

**Use nouns, not verbs:**
```
Good: /users, /orders, /products
Bad:  /getUsers, /createOrder, /fetchProducts
```

**Use plural nouns for collections:**
```
Good: /users, /orders
Bad:  /user, /order
```

**Use hierarchies for relationships:**
```
/users/123/orders          → Orders for user 123
/orders/456/items          → Items in order 456
/users/123/orders/456      → Order 456 for user 123
```

**Keep URLs lowercase:**
```
Good: /users/123/orders
Bad:  /Users/123/Orders
```

**Use hyphens for multi-word resources:**
```
Good: /product-categories, /order-items
Bad:  /productCategories, /product_categories
```

### Resource representations

A resource can have multiple representations. The same user might be represented as JSON, XML, or HTML:

```http
GET /users/123
Accept: application/json

{
  "id": "123",
  "name": "John"
}
```

```http
GET /users/123
Accept: application/xml

<user>
  <id>123</id>
  <name>John</name>
</user>
```

This is called **content negotiation**. The client specifies what format it wants using the `Accept` header.

---

## HTTP methods and their semantics

REST uses HTTP methods to indicate what operation to perform on a resource:

| Method | Purpose | Idempotent | Safe | Has Body |
|--------|---------|------------|------|----------|
| GET | Read a resource | Yes | Yes | No |
| POST | Create a resource | No | No | Yes |
| PUT | Replace a resource | Yes | No | Yes |
| PATCH | Partially update | No* | No | Yes |
| DELETE | Remove a resource | Yes | No | No |
| HEAD | Get headers only | Yes | Yes | No |
| OPTIONS | Get allowed methods | Yes | Yes | No |

**Idempotent:** Calling it multiple times has the same effect as calling it once.
**Safe:** Doesn't modify the resource.

*PATCH can be idempotent if designed carefully.

### GET - Read

Retrieves a resource. Should never modify data.

```http
GET /users/123 HTTP/1.1
Host: api.example.com

HTTP/1.1 200 OK
Content-Type: application/json

{
  "id": "123",
  "name": "John",
  "email": "john@example.com"
}
```

### POST - Create

Creates a new resource. The server assigns the ID.

```http
POST /users HTTP/1.1
Host: api.example.com
Content-Type: application/json

{
  "name": "John",
  "email": "john@example.com"
}

HTTP/1.1 201 Created
Location: /users/123
Content-Type: application/json

{
  "id": "123",
  "name": "John",
  "email": "john@example.com"
}
```

### PUT - Replace

Replaces an entire resource. If it doesn't exist, it might create it.

```http
PUT /users/123 HTTP/1.1
Host: api.example.com
Content-Type: application/json

{
  "name": "John Doe",
  "email": "johndoe@example.com"
}

HTTP/1.1 200 OK
```

**Important:** PUT should replace the entire resource. If you only send `name`, the `email` should be removed (or set to null).

### PATCH - Partial Update

Updates only the specified fields.

```http
PATCH /users/123 HTTP/1.1
Host: api.example.com
Content-Type: application/json

{
  "name": "John Doe"
}

HTTP/1.1 200 OK
```

Only `name` is updated. `email` stays the same.

### DELETE - Remove

Removes a resource.

```http
DELETE /users/123 HTTP/1.1
Host: api.example.com

HTTP/1.1 204 No Content
```

---

## Idempotency: Why it matters

An operation is idempotent if calling it multiple times has the same effect as calling it once.

**Why this matters:** Networks are unreliable. If a request times out, the client doesn't know if it succeeded. With idempotent operations, the client can safely retry.

```
Scenario: Client sends DELETE /users/123, times out

If DELETE is idempotent:
- Client retries DELETE /users/123
- Server returns 404 (already deleted) or 204 (deleted again)
- Either way, the user is deleted. No harm done.

If DELETE wasn't idempotent:
- Client retries DELETE /users/123
- Server might delete a different user, or throw an error
- Chaos ensues
```

### Making POST idempotent

POST is not naturally idempotent. Creating a user twice creates two users. But you can make it idempotent with idempotency keys:

```http
POST /orders HTTP/1.1
Idempotency-Key: abc123

{
  "product_id": "456",
  "quantity": 1
}
```

The server stores the idempotency key and result. If the same key is sent again, it returns the stored result instead of creating a new order.

---

## Richardson Maturity Model

Leonard Richardson defined a model for how "RESTful" an API is:

### Level 0: The Swamp of POX

One URL, one method (usually POST). Everything goes through it.

```http
POST /api HTTP/1.1

{
  "action": "getUser",
  "userId": "123"
}

POST /api HTTP/1.1

{
  "action": "createUser",
  "name": "John"
}
```

This is basically RPC over HTTP. Not REST at all.

### Level 1: Resources

Different URLs for different resources, but still using POST for everything.

```http
POST /users/123 HTTP/1.1
{
  "action": "get"
}

POST /users HTTP/1.1
{
  "action": "create",
  "name": "John"
}
```

Better, but still not using HTTP correctly.

### Level 2: HTTP Verbs

Using HTTP methods correctly.

```http
GET /users/123 HTTP/1.1

POST /users HTTP/1.1
{
  "name": "John"
}

DELETE /users/123 HTTP/1.1
```

This is where most "REST APIs" are. It's good enough for most use cases.

### Level 3: Hypermedia Controls (HATEOAS)

Responses include links to related resources and available actions.

```json
{
  "id": "123",
  "name": "John",
  "_links": {
    "self": {"href": "/users/123"},
    "orders": {"href": "/users/123/orders"},
    "delete": {"href": "/users/123", "method": "DELETE"}
  }
}
```

True REST. Rarely implemented in practice because it adds complexity.

---

## REST vs. RPC

REST and RPC (Remote Procedure Call) are different paradigms:

| Aspect | REST | RPC |
|--------|------|-----|
| Focus | Resources (nouns) | Actions (verbs) |
| URLs | `/users/123` | `/getUser`, `/createUser` |
| HTTP methods | GET, POST, PUT, DELETE | Usually just POST |
| Caching | Easy (GET is cacheable) | Hard |
| Discoverability | High (uniform interface) | Low |

**When to use REST:**
- Public APIs
- CRUD operations
- When caching matters
- When you want a standard interface

**When to use RPC:**
- Complex operations that don't map to CRUD
- Internal services where performance matters
- When you need streaming or bidirectional communication

---

## Common REST anti-patterns

### 1. Verbs in URLs

```
Bad:  GET /getUser/123
      POST /createUser
      POST /deleteUser/123

Good: GET /users/123
      POST /users
      DELETE /users/123
```

### 2. Using POST for everything

```
Bad:  POST /users/123/get
      POST /users/123/update
      POST /users/123/delete

Good: GET /users/123
      PUT /users/123
      DELETE /users/123
```

### 3. Ignoring HTTP status codes

```
Bad:  
HTTP/1.1 200 OK
{
  "success": false,
  "error": "User not found"
}

Good:
HTTP/1.1 404 Not Found
{
  "error": "User not found"
}
```

### 4. Not using proper content types

```
Bad:
POST /users
Content-Type: text/plain

{"name": "John"}

Good:
POST /users
Content-Type: application/json

{"name": "John"}
```

### 5. Deeply nested URLs

```
Bad:  /companies/123/departments/456/teams/789/members/012/tasks/345

Good: /tasks/345
      (with query params or body to filter by team/department if needed)
```

---

## Practical REST design tips

### 1. Start with resources, not endpoints

Don't think "I need an endpoint to get user orders." Think "I have a User resource and an Order resource. How are they related?"

### 2. Use query parameters for filtering, not different endpoints

```
Bad:
GET /active-users
GET /inactive-users
GET /users-created-today

Good:
GET /users?status=active
GET /users?status=inactive
GET /users?created_after=2024-01-15
```

### 3. Use sub-resources for relationships

```
GET /users/123/orders      → Orders for user 123
GET /orders/456/items      → Items in order 456
POST /users/123/addresses  → Add address to user 123
```

### 4. Handle actions that don't fit CRUD

Sometimes you need actions that don't map cleanly to CRUD. Options:

**Option 1: Treat the action as a resource**
```
POST /users/123/password-reset
POST /orders/456/cancellation
```

**Option 2: Use a controller resource**
```
POST /password-resets
{
  "user_id": "123"
}
```

**Option 3: Use query parameters**
```
POST /orders/456?action=cancel
```

I prefer Option 1 for most cases.

### 5. Be consistent with your conventions

Pick conventions and stick to them:

| Decision | My preference |
|----------|---------------|
| Naming | `snake_case` for JSON fields |
| Pluralization | Always plural (`/users`, not `/user`) |
| Trailing slashes | No trailing slash (`/users`, not `/users/`) |
| Case | Lowercase URLs |
| Dates | ISO 8601 (`2024-01-15T10:30:00Z`) |

---

## Key takeaways

1. **REST is an architectural style**, not a protocol. Most "REST APIs" are really just HTTP APIs.

2. **Statelessness is key** for scalability. Each request should contain everything needed to process it.

3. **Resources are nouns**, operations are HTTP methods. Don't put verbs in URLs.

4. **Idempotency matters** for reliability. GET, PUT, DELETE should be idempotent.

5. **Level 2 of Richardson Maturity Model** (resources + HTTP verbs) is good enough for most APIs.

6. **Consistency beats perfection**. Pick conventions and stick to them.

---

[Next: URL Design →](./03-url-design.md)