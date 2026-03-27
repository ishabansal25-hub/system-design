# What is API Design?

[← Back to API Design](./README.md)

---

Let's start with the basics. An API (Application Programming Interface) is a contract between two pieces of software. It defines how they talk to each other - what you can ask for, how you ask for it, and what you'll get back.

API design is the art of crafting that contract so it's:
- **Easy to understand** - developers can figure it out without reading a novel
- **Hard to misuse** - common mistakes are prevented by the design itself
- **Flexible enough** to evolve without breaking existing consumers
- **Consistent** - patterns repeat, so learning one part teaches you the rest

---

## Why does API design matter?

Here's the thing: once you publish an API, you're stuck with it. Unlike internal code that you can refactor whenever you want, APIs have consumers. Real people (or their code) depend on your endpoints working exactly as documented.

**Bad API design costs real money:**
- Developer time wasted figuring out confusing endpoints
- Support tickets from frustrated integrators
- Technical debt when you can't change things without breaking clients
- Lost customers who choose a competitor with a better developer experience

**Good API design pays dividends:**
- Faster integration times
- Fewer support requests
- Easier maintenance and evolution
- Happy developers who recommend your API to others

---

## What makes an API "good"?

I've used hundreds of APIs over the years. The good ones share these traits:

### 1. Predictability

If I know how to get a user (`GET /users/123`), I should be able to guess how to get an order (`GET /orders/456`). The patterns should be consistent.

```
Good:
GET /users/123
GET /orders/456
GET /products/789

Bad:
GET /users/123
GET /order?id=456
GET /fetchProduct/789
```

### 2. Self-documenting URLs

I should understand what an endpoint does just by looking at it.

```
Good:
POST /users/123/orders          → Create an order for user 123
GET /orders/456/items           → Get items in order 456
DELETE /users/123/addresses/1   → Delete address 1 for user 123

Bad:
POST /createNewOrder?user=123
GET /getOrderItems/456
DELETE /removeAddr?user=123&addr=1
```

### 3. Appropriate use of HTTP

HTTP has methods (GET, POST, PUT, DELETE) and status codes (200, 404, 500) for a reason. Use them correctly.

```
Good:
GET /users/123     → 200 OK with user data
GET /users/999     → 404 Not Found
POST /users        → 201 Created with new user
DELETE /users/123  → 204 No Content

Bad:
GET /users/123     → 200 OK with user data
GET /users/999     → 200 OK with {"error": "not found"}
POST /users        → 200 OK with new user
DELETE /users/123  → 200 OK with {"success": true}
```

### 4. Helpful error messages

When something goes wrong, tell me what and why.

```
Good:
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "The request body is invalid",
    "details": [
      {
        "field": "email",
        "message": "Must be a valid email address"
      },
      {
        "field": "age",
        "message": "Must be a positive integer"
      }
    ]
  }
}

Bad:
{
  "error": "Bad request"
}
```

### 5. Versioning from day one

You will make breaking changes. Plan for it.

```
Good:
GET /v1/users/123
GET /v2/users/123  (new version with different response format)

Bad:
GET /users/123     (no version, can't evolve without breaking clients)
```

---

## The different types of APIs

Not all APIs are the same. Here's a quick overview:

### REST (Representational State Transfer)

The most common type for web APIs. Uses HTTP methods and URLs to represent resources.

```
GET    /users/123      → Read user 123
POST   /users          → Create a new user
PUT    /users/123      → Replace user 123
PATCH  /users/123      → Update parts of user 123
DELETE /users/123      → Delete user 123
```

**Best for:** Public APIs, web applications, mobile backends

### GraphQL

A query language that lets clients request exactly the data they need.

```graphql
query {
  user(id: "123") {
    name
    email
    orders(last: 5) {
      id
      total
    }
  }
}
```

**Best for:** Complex data requirements, mobile apps with bandwidth constraints

### gRPC

A high-performance RPC framework using Protocol Buffers.

```protobuf
service UserService {
  rpc GetUser(GetUserRequest) returns (User);
  rpc CreateUser(CreateUserRequest) returns (User);
}
```

**Best for:** Internal microservices, high-throughput systems

### WebSockets

Persistent connections for real-time, bidirectional communication.

```javascript
socket.on('message', (data) => {
  console.log('Received:', data);
});
socket.send('Hello server!');
```

**Best for:** Chat applications, live updates, gaming

---

## API design principles

These principles apply regardless of which type of API you're building:

### 1. Design for your consumers, not your implementation

Your API should make sense to the people using it, not mirror your database schema.

```
Your database:
- users table
- user_addresses table (foreign key to users)
- user_preferences table (foreign key to users)

Bad API (exposes database structure):
GET /users/123
GET /user_addresses?user_id=123
GET /user_preferences?user_id=123

Good API (designed for consumers):
GET /users/123
{
  "id": "123",
  "name": "John",
  "addresses": [...],
  "preferences": {...}
}
```

### 2. Be consistent

Pick conventions and stick to them:

| Decision | Pick one |
|----------|----------|
| Naming | `snake_case` or `camelCase` |
| Pluralization | `/users` or `/user` |
| IDs | `id` or `userId` or `user_id` |
| Dates | ISO 8601 (`2024-01-15T10:30:00Z`) |
| Pagination | `page/limit` or `offset/limit` or cursors |

### 3. Make it hard to misuse

Good APIs prevent mistakes:

```
Bad: Accepts any string for email
POST /users
{"email": "not-an-email"}
→ 200 OK (creates user with invalid email)

Good: Validates input
POST /users
{"email": "not-an-email"}
→ 400 Bad Request
{
  "error": "email must be a valid email address"
}
```

### 4. Plan for failure

Networks fail. Servers crash. Design for it:

- **Idempotency**: Retrying a request shouldn't cause duplicate side effects
- **Timeouts**: Clients should know how long to wait
- **Retries**: Use exponential backoff with jitter
- **Circuit breakers**: Fail fast when downstream services are down

### 5. Think about evolution

Your API will change. Design for it:

- Version from day one
- Add fields, don't remove them
- Deprecate before removing
- Communicate changes clearly

---

## The API design process

Here's how I approach designing a new API:

### Step 1: Understand the use cases

Before writing any code, understand:
- Who will use this API?
- What are they trying to accomplish?
- What data do they need?
- What actions do they need to perform?

### Step 2: Identify the resources

List the "things" your API deals with:
- Users
- Orders
- Products
- Payments

### Step 3: Define the operations

For each resource, what can you do with it?
- Create, Read, Update, Delete?
- Any special actions?

### Step 4: Design the URLs

Map resources and operations to URLs:

```
Users:
  GET    /users          → List users
  POST   /users          → Create user
  GET    /users/{id}     → Get user
  PUT    /users/{id}     → Update user
  DELETE /users/{id}     → Delete user

User's orders:
  GET    /users/{id}/orders     → List user's orders
  POST   /users/{id}/orders     → Create order for user
```

### Step 5: Define request/response formats

What goes in, what comes out:

```
POST /users
Request:
{
  "name": "John Doe",
  "email": "john@example.com"
}

Response (201 Created):
{
  "id": "123",
  "name": "John Doe",
  "email": "john@example.com",
  "created_at": "2024-01-15T10:30:00Z"
}
```

### Step 6: Document and iterate

Write documentation, get feedback, improve. Repeat.

---

## Common mistakes to avoid

**1. Exposing internal implementation details**
```
Bad:  GET /api/v1/mysql/users/select?columns=name,email
Good: GET /api/v1/users/123
```

**2. Using verbs in URLs**
```
Bad:  POST /createUser, GET /getUser/123
Good: POST /users, GET /users/123
```

**3. Ignoring HTTP semantics**
```
Bad:  POST /users/123/delete
Good: DELETE /users/123
```

**4. Inconsistent naming**
```
Bad:  /users/123, /Orders/456, /product-items/789
Good: /users/123, /orders/456, /product-items/789
```

**5. Poor error handling**
```
Bad:  200 OK with {"error": "something went wrong"}
Good: 400 Bad Request with detailed error message
```

---

## Measuring API quality

How do you know if your API is good? Track these:

| Metric | What it tells you |
|--------|-------------------|
| Time to first call | How quickly can a new developer make their first successful request? |
| Support tickets | How many questions do developers have? |
| Error rates | Are developers making mistakes? |
| Adoption | Are people actually using it? |
| Documentation views | What are people confused about? |

---

## Key takeaways

1. **APIs are contracts** - once published, they're hard to change
2. **Design for consumers** - not your database or implementation
3. **Consistency is king** - patterns should repeat throughout
4. **Errors are features** - helpful error messages save everyone time
5. **Plan for evolution** - version from day one
6. **Test with real users** - get feedback early and often

---

[Next: REST Fundamentals →](./02-rest-fundamentals.md)