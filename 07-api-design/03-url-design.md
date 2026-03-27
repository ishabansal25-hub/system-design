# URL Design

[← Back to API Design](./README.md) | [← Previous: REST Fundamentals](./02-rest-fundamentals.md)

---

URLs are the front door to your API. A well-designed URL tells developers what they're going to get before they even read the documentation. A poorly designed one makes them guess, check the docs, and probably still get it wrong.

I've seen APIs where I had to read a 50-page PDF just to understand the URL structure. Don't be that API.

---

## The anatomy of a URL

Let's break down a URL:

```
https://api.example.com/v1/users/123/orders?status=pending&limit=10
└─┬──┘ └──────┬───────┘└┬┘└────────┬───────┘└──────────┬──────────┘
scheme      host     version    path              query string
```

| Component | Purpose | Example |
|-----------|---------|---------|
| Scheme | Protocol | `https` |
| Host | Server address | `api.example.com` |
| Version | API version | `/v1` |
| Path | Resource location | `/users/123/orders` |
| Query string | Filtering, pagination | `?status=pending&limit=10` |

---

## URL design principles

### 1. Use nouns, not verbs

URLs identify resources (things), not actions. Actions are expressed through HTTP methods.

```
Bad:
GET  /getUsers
POST /createUser
POST /deleteUser/123
GET  /fetchUserOrders/123

Good:
GET    /users           → Get all users
POST   /users           → Create a user
DELETE /users/123       → Delete user 123
GET    /users/123/orders → Get orders for user 123
```

The HTTP method tells you the action. The URL tells you what you're acting on.

### 2. Use plural nouns for collections

Be consistent. Always use plurals.

```
Bad:
/user/123
/order/456
/product/789

Good:
/users/123
/orders/456
/products/789
```

Why? Because `/users` is a collection, and `/users/123` is an item in that collection. It reads naturally: "from the users collection, get user 123."

### 3. Use lowercase letters

URLs are case-sensitive (technically). Avoid confusion by always using lowercase.

```
Bad:
/Users/123
/productCategories
/Order_Items

Good:
/users/123
/product-categories
/order-items
```

### 4. Use hyphens for readability

When you need multiple words, use hyphens. Not underscores, not camelCase.

```
Bad:
/productCategories
/product_categories
/ProductCategories

Good:
/product-categories
```

Why hyphens? They're more readable and SEO-friendly. Google treats hyphens as word separators but not underscores.

### 5. Don't include file extensions

The response format should be determined by the `Accept` header, not the URL.

```
Bad:
/users/123.json
/users/123.xml
/reports/monthly.pdf

Good:
/users/123
Accept: application/json

/users/123
Accept: application/xml

/reports/monthly
Accept: application/pdf
```

### 6. Keep URLs short but meaningful

Every segment should add meaning. If it doesn't, remove it.

```
Bad:
/api/v1/services/user-service/resources/users/123

Good:
/v1/users/123
```

---

## Resource hierarchies

URLs should reflect the relationships between resources.

### Parent-child relationships

When one resource belongs to another, nest it:

```
/users/123/orders          → Orders belonging to user 123
/orders/456/items          → Items in order 456
/teams/789/members         → Members of team 789
```

### When to nest vs. when not to

**Nest when:**
- The child resource doesn't make sense without the parent
- You always need the parent context to access the child
- The relationship is truly hierarchical

**Don't nest when:**
- The child resource has its own identity
- You need to access the child directly
- The nesting would be more than 2-3 levels deep

```
Good nesting:
/users/123/addresses       → User's addresses (addresses belong to users)
/orders/456/line-items     → Order's line items (items belong to orders)

Avoid deep nesting:
Bad:  /companies/1/departments/2/teams/3/members/4/tasks/5
Good: /tasks/5
      /tasks?team_id=3
      /teams/3/tasks
```

### Accessing nested resources directly

Sometimes you need both:

```
/users/123/orders          → All orders for user 123
/orders/456                → Order 456 directly (when you know the ID)
```

This is fine. The same resource can be accessible through multiple URLs.

---

## Query parameters

Query parameters are for filtering, sorting, pagination, and optional modifiers. They don't identify resources.

### Filtering

```
GET /users?status=active
GET /users?role=admin&status=active
GET /orders?created_after=2024-01-01
GET /products?category=electronics&price_max=1000
```

### Sorting

```
GET /users?sort=created_at
GET /users?sort=-created_at          → Descending (prefix with -)
GET /users?sort=last_name,first_name → Multiple fields
GET /products?sort=-price,name       → Price desc, then name asc
```

### Pagination

```
GET /users?page=2&limit=20
GET /users?offset=40&limit=20
GET /users?cursor=eyJpZCI6MTIzfQ==
```

### Field selection

```
GET /users?fields=id,name,email
GET /users/123?fields=id,name,email
```

### Expansion/embedding

```
GET /orders/123?expand=customer,items
GET /users/123?include=orders,addresses
```

### Search

```
GET /users?q=john
GET /products?search=laptop
```

---

## Common URL patterns

### Collection operations

```
GET    /users              → List all users
POST   /users              → Create a new user
DELETE /users              → Delete all users (use with caution!)
```

### Item operations

```
GET    /users/123          → Get user 123
PUT    /users/123          → Replace user 123
PATCH  /users/123          → Update user 123
DELETE /users/123          → Delete user 123
```

### Sub-resource operations

```
GET    /users/123/orders   → List user's orders
POST   /users/123/orders   → Create order for user
GET    /users/123/orders/456 → Get specific order
```

### Actions that don't fit CRUD

Sometimes you need to perform actions that aren't simple CRUD. Here are patterns:

**Pattern 1: Action as a sub-resource**
```
POST /users/123/activate
POST /users/123/deactivate
POST /orders/456/cancel
POST /orders/456/refund
POST /users/123/password-reset
```

**Pattern 2: Action as a resource**
```
POST /activations
{
  "user_id": "123"
}

POST /cancellations
{
  "order_id": "456",
  "reason": "Customer request"
}
```

**Pattern 3: State as a resource**
```
PUT /users/123/status
{
  "status": "active"
}

PUT /orders/456/state
{
  "state": "cancelled"
}
```

I prefer Pattern 1 for simple actions and Pattern 2 when the action needs to be tracked as its own entity.

---

## Versioning in URLs

There are several approaches to API versioning. URL versioning is the most common:

### URL path versioning

```
/v1/users/123
/v2/users/123
```

**Pros:**
- Very explicit
- Easy to see which version you're using
- Easy to route at the infrastructure level

**Cons:**
- Not "pure" REST (version isn't part of the resource)
- Can lead to URL proliferation

### Subdomain versioning

```
https://v1.api.example.com/users/123
https://v2.api.example.com/users/123
```

**Pros:**
- Clean URLs
- Easy to route

**Cons:**
- More complex DNS setup
- CORS can be tricky

### Query parameter versioning

```
/users/123?version=1
/users/123?api-version=2024-01-15
```

**Pros:**
- Doesn't change the URL structure
- Can default to latest version

**Cons:**
- Easy to forget
- Caching can be tricky

### Header versioning

```
GET /users/123
Accept: application/vnd.example.v1+json
```

or

```
GET /users/123
X-API-Version: 1
```

**Pros:**
- "Pure" REST
- Clean URLs

**Cons:**
- Not visible in the URL
- Harder to test in browser
- Easy to forget

### My recommendation

Use URL path versioning (`/v1/users/123`). It's the most explicit and widely understood. Save the "purity" debates for academic papers.

---

## Real-world URL examples

Let's look at how some well-known APIs design their URLs:

### Stripe

```
GET  /v1/customers
POST /v1/customers
GET  /v1/customers/cus_123
POST /v1/customers/cus_123

GET  /v1/charges
POST /v1/charges
GET  /v1/charges/ch_456

POST /v1/refunds
GET  /v1/refunds/re_789
```

Notice: Flat structure, prefixed IDs, actions as resources (refunds).

### GitHub

```
GET  /users/octocat
GET  /users/octocat/repos
GET  /repos/octocat/hello-world
GET  /repos/octocat/hello-world/issues
POST /repos/octocat/hello-world/issues
GET  /repos/octocat/hello-world/issues/1
```

Notice: Hierarchical, uses usernames in URLs, no version in URL (uses Accept header).

### Twitter (X)

```
GET  /2/users/123
GET  /2/users/by/username/jack
GET  /2/tweets/456
POST /2/tweets
GET  /2/users/123/tweets
GET  /2/users/123/followers
```

Notice: Version in URL, multiple ways to access users (by ID or username).

### Twilio

```
GET  /2010-04-01/Accounts/AC123/Messages
POST /2010-04-01/Accounts/AC123/Messages
GET  /2010-04-01/Accounts/AC123/Messages/SM456
GET  /2010-04-01/Accounts/AC123/Calls
```

Notice: Date-based versioning, account scoping, PascalCase resources (unusual but consistent).

---

## URL design for specific scenarios

### Multi-tenant APIs

When your API serves multiple tenants (organizations, accounts):

**Option 1: Tenant in path**
```
/tenants/acme/users/123
/organizations/org_123/projects
```

**Option 2: Tenant in subdomain**
```
https://acme.api.example.com/users/123
```

**Option 3: Tenant in header**
```
GET /users/123
X-Tenant-ID: acme
```

I prefer Option 1 for public APIs (explicit) and Option 3 for internal APIs (cleaner URLs).

### Composite resources

When you need to return data from multiple resources:

```
GET /dashboard
GET /users/123/summary
GET /reports/sales-overview
```

These are "virtual" resources that aggregate data. They're fine as long as they're clearly named.

### Batch operations

When you need to operate on multiple resources at once:

**Option 1: Comma-separated IDs**
```
GET /users/123,456,789
DELETE /users/123,456,789
```

**Option 2: Query parameter**
```
GET /users?ids=123,456,789
DELETE /users?ids=123,456,789
```

**Option 3: Batch endpoint**
```
POST /users/batch
{
  "ids": ["123", "456", "789"],
  "action": "delete"
}
```

I prefer Option 2 for GET and Option 3 for mutations.

### Search endpoints

```
GET /search?q=laptop
GET /search/products?q=laptop
GET /products/search?q=laptop
POST /search
{
  "query": "laptop",
  "filters": {...}
}
```

For simple searches, use GET with query params. For complex searches with many filters, POST is acceptable.

---

## Common mistakes

### 1. Verbs in URLs

```
Bad:  /getUsers, /createOrder, /deleteProduct/123
Good: /users, /orders, /products/123
```

### 2. Inconsistent pluralization

```
Bad:  /user/123, /orders/456, /product/789
Good: /users/123, /orders/456, /products/789
```

### 3. Unnecessary nesting

```
Bad:  /users/123/orders/456/items/789/details
Good: /order-items/789
```

### 4. Exposing implementation details

```
Bad:  /api/mysql/tables/users/rows/123
Good: /users/123
```

### 5. Using IDs that expose information

```
Bad:  /users/1, /users/2, /users/3  (sequential, reveals user count)
Good: /users/a1b2c3d4  (random, opaque)
```

### 6. Inconsistent casing

```
Bad:  /users/123, /productCategories, /Order_Items
Good: /users/123, /product-categories, /order-items
```

### 7. Trailing slashes inconsistency

```
Bad:  /users/ and /orders (mixing)
Good: /users and /orders (consistent, no trailing slash)
```

---

## URL design checklist

Before finalizing your URLs, check:

- [ ] Are all URLs lowercase?
- [ ] Are hyphens used for multi-word resources?
- [ ] Are nouns used instead of verbs?
- [ ] Are collections plural?
- [ ] Is nesting limited to 2-3 levels?
- [ ] Are query parameters used for filtering/sorting/pagination?
- [ ] Is versioning consistent?
- [ ] Can developers guess the URL structure?
- [ ] Are IDs opaque (not sequential)?
- [ ] Is there no trailing slash (or consistent trailing slash)?

---

## Key takeaways

1. **URLs are for developers.** Design them to be guessable and self-documenting.

2. **Nouns, not verbs.** The HTTP method is the verb.

3. **Plural nouns for collections.** `/users`, not `/user`.

4. **Hyphens for readability.** `/product-categories`, not `/productCategories`.

5. **Keep nesting shallow.** 2-3 levels max.

6. **Query params for filtering.** Not different endpoints.

7. **Be consistent.** Whatever conventions you choose, apply them everywhere.

8. **Version in the URL.** It's the most explicit approach.

---

[Next: HTTP Methods Deep Dive →](./04-http-methods-deep-dive.md)