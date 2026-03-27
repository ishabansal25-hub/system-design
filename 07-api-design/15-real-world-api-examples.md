# Real-World API Examples

[← Back to API Design](./README.md) | [← Previous: API Gateway Patterns](./14-api-gateway-patterns.md)

---

The best way to learn API design is to study APIs that millions of developers use every day. These APIs have evolved through years of real-world usage, feedback, and iteration. Let's analyze what makes them great (and sometimes, what doesn't).

---

## Stripe API

Stripe is often cited as the gold standard for API design. Here's why.

### What they do well

**Consistent resource naming:**
```
/v1/customers
/v1/customers/{id}
/v1/customers/{id}/sources
/v1/charges
/v1/refunds
/v1/subscriptions
```

**Expandable responses:**
```bash
# Basic response
GET /v1/charges/ch_123

{
  "id": "ch_123",
  "customer": "cus_456"  # Just the ID
}

# Expanded response
GET /v1/charges/ch_123?expand[]=customer

{
  "id": "ch_123",
  "customer": {
    "id": "cus_456",
    "email": "john@example.com",
    "name": "John Doe"
  }
}
```

**Idempotency keys:**
```bash
POST /v1/charges
Idempotency-Key: unique-key-123

# Safe to retry - won't create duplicate charges
```

**Versioning via date:**
```bash
Stripe-Version: 2024-01-15
```

**Excellent error messages:**
```json
{
  "error": {
    "type": "card_error",
    "code": "card_declined",
    "message": "Your card was declined.",
    "param": "card_number",
    "decline_code": "insufficient_funds",
    "doc_url": "https://stripe.com/docs/error-codes/card-declined"
  }
}
```

### Key patterns

- Test mode with `sk_test_` keys
- Webhooks for async events
- Metadata on every object
- Pagination with cursors

---

## GitHub API

GitHub's API demonstrates how to handle complex resources and relationships.

### REST API

```bash
# Get a repository
GET /repos/octocat/hello-world

# Get repository issues
GET /repos/octocat/hello-world/issues

# Get a specific issue
GET /repos/octocat/hello-world/issues/1

# Get issue comments
GET /repos/octocat/hello-world/issues/1/comments
```

### Pagination with Link headers

```bash
GET /repos/octocat/hello-world/issues?per_page=30

HTTP/1.1 200 OK
Link: <https://api.github.com/...?page=2>; rel="next",
      <https://api.github.com/...?page=10>; rel="last"
```

### Conditional requests

```bash
# First request
GET /repos/octocat/hello-world
→ ETag: "abc123"

# Subsequent request
GET /repos/octocat/hello-world
If-None-Match: "abc123"
→ 304 Not Modified (use cached version)
```

### GraphQL API

```graphql
query {
  repository(owner: "octocat", name: "hello-world") {
    name
    description
    stargazerCount
    issues(first: 10, states: OPEN) {
      nodes {
        title
        author {
          login
        }
      }
    }
  }
}
```

---

## Twitter/X API

Twitter's API shows how to handle real-time data and high-volume operations.

### Tweet operations

```bash
# Get a tweet
GET /2/tweets/123456789

# Get user's tweets
GET /2/users/123/tweets

# Post a tweet
POST /2/tweets
{
  "text": "Hello, world!"
}
```

### Field selection

```bash
GET /2/tweets/123?tweet.fields=created_at,public_metrics&expansions=author_id&user.fields=name,username
```

### Streaming API

```bash
# Real-time filtered stream
GET /2/tweets/search/stream

# Returns tweets matching your rules in real-time
```

### Rate limiting

```
App-level: 300 requests / 15 minutes
User-level: 900 requests / 15 minutes

Headers:
x-rate-limit-limit: 900
x-rate-limit-remaining: 899
x-rate-limit-reset: 1234567890
```

---

## Twilio API

Twilio demonstrates excellent API design for communication services.

### Resource hierarchy

```bash
# Accounts contain everything
/2010-04-01/Accounts/{AccountSid}

# Messages under accounts
/2010-04-01/Accounts/{AccountSid}/Messages

# Calls under accounts
/2010-04-01/Accounts/{AccountSid}/Calls
```

### Sending an SMS

```bash
POST /2010-04-01/Accounts/{AccountSid}/Messages
Content-Type: application/x-www-form-urlencoded

To=+15551234567&From=+15559876543&Body=Hello!

HTTP/1.1 201 Created
{
  "sid": "SM123...",
  "status": "queued",
  "to": "+15551234567",
  "from": "+15559876543",
  "body": "Hello!"
}
```

### Webhooks for status updates

```bash
# Twilio calls your webhook when message status changes
POST /your-webhook
{
  "MessageSid": "SM123...",
  "MessageStatus": "delivered",
  "To": "+15551234567"
}
```

---

## Slack API

Slack shows how to build APIs for collaboration tools.

### Web API

```bash
# Post a message
POST /api/chat.postMessage
Content-Type: application/json
Authorization: Bearer xoxb-...

{
  "channel": "C1234567890",
  "text": "Hello, world!",
  "attachments": [...]
}
```

### Events API

```bash
# Slack sends events to your endpoint
POST /your-endpoint
{
  "type": "event_callback",
  "event": {
    "type": "message",
    "channel": "C1234567890",
    "user": "U1234567890",
    "text": "Hello!"
  }
}
```

### Interactive components

```bash
# User clicks a button, Slack sends:
POST /your-endpoint
{
  "type": "block_actions",
  "actions": [
    {
      "action_id": "approve_button",
      "value": "approve"
    }
  ]
}
```

---

## Shopify API

Shopify demonstrates e-commerce API patterns.

### REST Admin API

```bash
# Get products
GET /admin/api/2024-01/products.json

# Get a specific product
GET /admin/api/2024-01/products/123.json

# Create a product
POST /admin/api/2024-01/products.json
{
  "product": {
    "title": "Burton Custom Freestyle 151",
    "body_html": "<strong>Good snowboard!</strong>",
    "vendor": "Burton"
  }
}
```

### Cursor-based pagination

```bash
GET /admin/api/2024-01/products.json?limit=50

{
  "products": [...],
  "next_page_info": "eyJsYXN0X2lkIjo..."
}

# Next page
GET /admin/api/2024-01/products.json?page_info=eyJsYXN0X2lkIjo...
```

### GraphQL Admin API

```graphql
mutation {
  productCreate(input: {
    title: "New Product",
    productType: "Snowboard",
    vendor: "Burton"
  }) {
    product {
      id
      title
    }
    userErrors {
      field
      message
    }
  }
}
```

---

## Common patterns across great APIs

### 1. Consistent naming

```
✓ /users, /orders, /products (plural nouns)
✓ /users/{id}/orders (nested resources)
✓ snake_case or camelCase (pick one, stick to it)
```

### 2. Predictable responses

```json
// Success
{
  "data": {...},
  "meta": {...}
}

// Error
{
  "error": {
    "code": "...",
    "message": "..."
  }
}
```

### 3. Helpful error messages

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "The email field must be a valid email address",
    "field": "email",
    "documentation_url": "https://..."
  }
}
```

### 4. Versioning strategy

```
Stripe: Date-based headers (Stripe-Version: 2024-01-15)
GitHub: URL path (/v3/)
Twilio: URL path with date (/2010-04-01/)
Shopify: URL path with date (/admin/api/2024-01/)
```

### 5. Rate limiting transparency

```
X-RateLimit-Limit: 5000
X-RateLimit-Remaining: 4999
X-RateLimit-Reset: 1234567890
```

### 6. Webhooks for async operations

```
All major APIs use webhooks:
- Stripe: payment events
- GitHub: repository events
- Twilio: message status
- Slack: user interactions
```

### 7. SDKs in multiple languages

```
Stripe: Ruby, Python, PHP, Java, Node.js, Go, .NET
GitHub: JavaScript, Ruby, Python, and more
Twilio: 7+ languages
```

---

## Anti-patterns to avoid

### 1. Inconsistent naming

```
Bad:
/getUsers
/user/create
/orders_list
/DeleteProduct

Good:
/users
/users
/orders
/products/{id}
```

### 2. Verbs in URLs

```
Bad:
/users/123/activate
/orders/456/cancel

Better:
POST /users/123/activation
POST /orders/456/cancellation

Or:
PATCH /users/123 {"status": "active"}
PATCH /orders/456 {"status": "cancelled"}
```

### 3. Exposing internal IDs

```
Bad:
/users/1  (sequential, guessable)

Good:
/users/usr_a1b2c3d4  (prefixed, random)
```

### 4. Returning 200 for errors

```
Bad:
HTTP/1.1 200 OK
{"success": false, "error": "Not found"}

Good:
HTTP/1.1 404 Not Found
{"error": {"code": "NOT_FOUND", "message": "User not found"}}
```

### 5. Breaking changes without versioning

```
Bad:
Rename "name" to "full_name" in production

Good:
Add "full_name", deprecate "name", remove in next version
```

---

## Key takeaways

1. **Study the best.** Stripe, GitHub, Twilio - they've solved problems you'll face.

2. **Consistency is king.** Pick patterns and stick to them.

3. **Errors should help.** Include codes, messages, and documentation links.

4. **Plan for scale.** Pagination, rate limiting, webhooks from day one.

5. **Version thoughtfully.** Breaking changes happen - handle them gracefully.

6. **Provide SDKs.** Make integration as easy as possible.

7. **Document everything.** Great APIs have great documentation.

---

[Next: Interview Cheatsheet →](./INTERVIEW-CHEATSHEET.md)