# Pagination, Filtering & Sorting

[← Back to API Design](./README.md) | [← Previous: Rate Limiting & Throttling](./09-rate-limiting-throttling.md)

---

When your API returns collections, you need to handle three things: pagination (how much data to return), filtering (which data to return), and sorting (in what order). Get these wrong, and you'll have slow APIs, frustrated developers, and crashed databases.

I've seen APIs that returned 100,000 records in a single response. The client crashed. The server crashed. Everyone had a bad day.

---

## Pagination

### Why paginate?

```
Without pagination:
GET /users → Returns 1,000,000 users
- Server: Out of memory
- Network: Timeout
- Client: Crashes trying to parse

With pagination:
GET /users?limit=20 → Returns 20 users
- Server: Fast query
- Network: Small payload
- Client: Renders quickly
```

### Pagination strategies

#### 1. Offset-based pagination

The most common approach. Specify offset (skip) and limit (take).

```http
GET /users?offset=0&limit=20   → Users 1-20
GET /users?offset=20&limit=20  → Users 21-40
GET /users?offset=40&limit=20  → Users 41-60
```

Or using page numbers:
```http
GET /users?page=1&limit=20     → Users 1-20
GET /users?page=2&limit=20     → Users 21-40
GET /users?page=3&limit=20     → Users 41-60
```

**Response:**
```json
{
  "data": [
    {"id": "1", "name": "Alice"},
    {"id": "2", "name": "Bob"},
    ...
  ],
  "pagination": {
    "total": 1000,
    "page": 1,
    "limit": 20,
    "pages": 50,
    "has_next": true,
    "has_prev": false
  }
}
```

**Pros:**
- Simple to implement
- Easy to jump to any page
- Familiar to developers

**Cons:**
- Slow for large offsets (`OFFSET 1000000` is expensive)
- Inconsistent results if data changes between requests
- Can miss or duplicate items

```
Problem: Data changes between requests

Page 1: [A, B, C, D, E] (offset=0)
-- New item X inserted at position 1 --
Page 2: [E, F, G, H, I] (offset=5)

Item E appears twice! Item X is missed!
```

**SQL:**
```sql
SELECT * FROM users
ORDER BY created_at DESC
LIMIT 20 OFFSET 40;
```

#### 2. Cursor-based pagination (keyset pagination)

Use a cursor (pointer) to the last item, fetch items after it.

```http
GET /users?limit=20
→ Returns users 1-20, cursor="eyJpZCI6MjB9"

GET /users?limit=20&cursor=eyJpZCI6MjB9
→ Returns users 21-40, cursor="eyJpZCI6NDB9"

GET /users?limit=20&cursor=eyJpZCI6NDB9
→ Returns users 41-60, cursor="eyJpZCI6NjB9"
```

**Response:**
```json
{
  "data": [
    {"id": "21", "name": "User 21"},
    {"id": "22", "name": "User 22"},
    ...
  ],
  "pagination": {
    "next_cursor": "eyJpZCI6NDB9",
    "prev_cursor": "eyJpZCI6MjB9",
    "has_next": true,
    "has_prev": true
  }
}
```

**Cursor encoding:**
```python
# Cursor is typically base64-encoded JSON
import base64
import json

def encode_cursor(last_item):
    data = {"id": last_item.id, "created_at": last_item.created_at}
    return base64.b64encode(json.dumps(data).encode()).decode()

def decode_cursor(cursor):
    return json.loads(base64.b64decode(cursor).decode())
```

**Pros:**
- Consistent results even if data changes
- Fast for any position (no OFFSET)
- Scales to millions of records

**Cons:**
- Can't jump to arbitrary page
- More complex to implement
- Cursor can become invalid if referenced item is deleted

**SQL:**
```sql
-- Instead of OFFSET
SELECT * FROM users
WHERE (created_at, id) < ('2024-01-15 10:30:00', 40)
ORDER BY created_at DESC, id DESC
LIMIT 20;
```

#### 3. Time-based pagination

Use timestamps as cursors.

```http
GET /events?since=2024-01-15T10:00:00Z&limit=100
GET /events?since=2024-01-15T11:00:00Z&limit=100
```

**Pros:**
- Natural for time-series data
- Easy to understand
- Good for real-time feeds

**Cons:**
- Requires timestamp field
- Can miss items with same timestamp

#### Comparison

| Aspect | Offset | Cursor | Time-based |
|--------|--------|--------|------------|
| Jump to page | ✅ | ❌ | ❌ |
| Performance at scale | ❌ | ✅ | ✅ |
| Consistent results | ❌ | ✅ | ✅ |
| Complexity | Low | Medium | Low |
| Use case | Small datasets, UI tables | Large datasets, feeds | Time-series, logs |

**My recommendation:** 
- Use cursor-based for large datasets and feeds
- Use offset-based for admin UIs and small datasets
- Use time-based for logs and events

---

## Filtering

### Query parameter filtering

```http
GET /users?status=active
GET /users?role=admin&status=active
GET /products?category=electronics&price_min=100&price_max=500
GET /orders?created_after=2024-01-01&created_before=2024-01-31
```

### Common filter patterns

**Equality:**
```http
GET /users?status=active
GET /users?role=admin
```

**Comparison:**
```http
GET /products?price_min=100
GET /products?price_max=500
GET /products?price_gte=100&price_lte=500
GET /orders?created_after=2024-01-01
```

**Multiple values (OR):**
```http
GET /users?status=active,pending
GET /products?category=electronics,computers
```

**Negation:**
```http
GET /users?status_not=deleted
GET /products?category_ne=accessories
```

**Pattern matching:**
```http
GET /users?name_contains=john
GET /users?email_starts_with=admin
GET /products?name_like=*phone*
```

**Null checks:**
```http
GET /users?deleted_at_null=true
GET /orders?shipped_at_not_null=true
```

### Filter syntax options

**Option 1: Flat parameters (simple)**
```http
GET /products?category=electronics&price_min=100&price_max=500
```

**Option 2: Bracket notation (more expressive)**
```http
GET /products?filter[category]=electronics&filter[price][gte]=100&filter[price][lte]=500
```

**Option 3: JSON filter (complex queries)**
```http
GET /products?filter={"category":"electronics","price":{"$gte":100,"$lte":500}}
```

**Option 4: Query language (very complex)**
```http
GET /products?q=category:electronics AND price:[100 TO 500]
```

**My recommendation:** Start with flat parameters. Move to bracket notation if you need more expressiveness.

### Filter response

Include applied filters in response:

```json
{
  "data": [...],
  "filters": {
    "status": "active",
    "role": "admin"
  },
  "pagination": {...}
}
```

---

## Sorting

### Basic sorting

```http
GET /users?sort=name           → Sort by name ascending
GET /users?sort=-name          → Sort by name descending
GET /users?sort=created_at     → Sort by created_at ascending
GET /users?sort=-created_at    → Sort by created_at descending
```

### Multiple sort fields

```http
GET /users?sort=last_name,first_name
GET /users?sort=-created_at,name
GET /products?sort=-rating,-reviews_count,price
```

### Sort syntax options

**Option 1: Prefix notation (common)**
```http
GET /users?sort=-created_at    → Descending
GET /users?sort=created_at     → Ascending
```

**Option 2: Explicit direction**
```http
GET /users?sort=created_at&order=desc
GET /users?sort_by=created_at&sort_order=desc
```

**Option 3: Bracket notation**
```http
GET /users?sort[created_at]=desc&sort[name]=asc
```

**My recommendation:** Prefix notation (`-` for descending) is concise and widely used.

### Sort response

Include sort info in response:

```json
{
  "data": [...],
  "sort": [
    {"field": "created_at", "direction": "desc"},
    {"field": "name", "direction": "asc"}
  ],
  "pagination": {...}
}
```

---

## Combining pagination, filtering, and sorting

```http
GET /products?category=electronics&price_min=100&sort=-rating&limit=20&page=1
```

**Response:**
```json
{
  "data": [
    {
      "id": "prod_123",
      "name": "Smartphone X",
      "category": "electronics",
      "price": 599.99,
      "rating": 4.8
    },
    ...
  ],
  "filters": {
    "category": "electronics",
    "price_min": 100
  },
  "sort": [
    {"field": "rating", "direction": "desc"}
  ],
  "pagination": {
    "total": 150,
    "page": 1,
    "limit": 20,
    "pages": 8,
    "has_next": true,
    "has_prev": false
  }
}
```

---

## Field selection

Let clients request only the fields they need:

```http
GET /users?fields=id,name,email
GET /users/123?fields=id,name,email,created_at
```

**Response:**
```json
{
  "data": [
    {
      "id": "123",
      "name": "John Doe",
      "email": "john@example.com"
    }
  ]
}
```

### Nested field selection

```http
GET /orders?fields=id,total,customer.name,customer.email
GET /orders?fields=id,items.name,items.price
```

### Excluding fields

```http
GET /users?exclude=password,internal_notes
```

---

## Search

### Simple search

```http
GET /users?q=john
GET /products?search=laptop
```

### Search with filters

```http
GET /products?q=laptop&category=electronics&price_max=1000
```

### Full-text search response

```json
{
  "data": [
    {
      "id": "prod_123",
      "name": "Gaming Laptop Pro",
      "description": "High-performance laptop for gaming...",
      "_score": 0.95,
      "_highlights": {
        "name": "Gaming <em>Laptop</em> Pro",
        "description": "High-performance <em>laptop</em> for gaming..."
      }
    }
  ],
  "search": {
    "query": "laptop",
    "total_results": 150
  }
}
```

---

## Implementation considerations

### Database performance

**Offset pagination:**
```sql
-- Slow for large offsets
SELECT * FROM users ORDER BY id LIMIT 20 OFFSET 1000000;
-- Database must scan 1,000,020 rows!
```

**Cursor pagination:**
```sql
-- Fast regardless of position
SELECT * FROM users WHERE id > 1000000 ORDER BY id LIMIT 20;
-- Database uses index, scans only 20 rows
```

### Indexing for filters and sorts

```sql
-- Create indexes for common filter/sort combinations
CREATE INDEX idx_users_status ON users(status);
CREATE INDEX idx_users_created_at ON users(created_at DESC);
CREATE INDEX idx_users_status_created ON users(status, created_at DESC);

-- For cursor pagination
CREATE INDEX idx_users_cursor ON users(created_at DESC, id DESC);
```

### Limiting filter options

Don't let users filter on unindexed columns:

```python
ALLOWED_FILTERS = ['status', 'role', 'created_at', 'email']
ALLOWED_SORTS = ['created_at', 'name', 'email']

def validate_filters(filters):
    for key in filters:
        if key not in ALLOWED_FILTERS:
            raise BadRequest(f"Cannot filter by '{key}'")

def validate_sort(sort_field):
    if sort_field.lstrip('-') not in ALLOWED_SORTS:
        raise BadRequest(f"Cannot sort by '{sort_field}'")
```

### Default limits

Always have defaults and maximums:

```python
DEFAULT_LIMIT = 20
MAX_LIMIT = 100

def get_limit(request):
    limit = request.args.get('limit', DEFAULT_LIMIT)
    return min(int(limit), MAX_LIMIT)
```

---

## Response structure patterns

### Envelope pattern

```json
{
  "data": [...],
  "pagination": {...},
  "filters": {...},
  "sort": {...}
}
```

### Links pattern (HATEOAS)

```json
{
  "data": [...],
  "links": {
    "self": "/users?page=2&limit=20",
    "first": "/users?page=1&limit=20",
    "prev": "/users?page=1&limit=20",
    "next": "/users?page=3&limit=20",
    "last": "/users?page=50&limit=20"
  },
  "meta": {
    "total": 1000,
    "page": 2,
    "limit": 20
  }
}
```

### Cursor pattern

```json
{
  "data": [...],
  "cursors": {
    "before": "eyJpZCI6MjB9",
    "after": "eyJpZCI6NDB9"
  },
  "has_more": true
}
```

---

## Real-world examples

### GitHub

```http
GET /repos/owner/repo/issues?state=open&sort=created&direction=desc&per_page=30&page=2

Link: <https://api.github.com/...?page=3>; rel="next",
      <https://api.github.com/...?page=1>; rel="prev",
      <https://api.github.com/...?page=1>; rel="first",
      <https://api.github.com/...?page=10>; rel="last"
```

### Stripe

```http
GET /v1/customers?limit=10&starting_after=cus_123

{
  "data": [...],
  "has_more": true,
  "url": "/v1/customers"
}
```

### Shopify

```http
GET /admin/api/2024-01/products.json?limit=50&since_id=123&fields=id,title,price

{
  "products": [...],
  "next_page_info": "eyJsYXN0X2lkIjo..."
}
```

---

## Best practices

### 1. Always paginate collections

```python
# Never return unbounded results
@app.route('/users')
def get_users():
    limit = min(request.args.get('limit', 20), 100)
    # Always apply limit
```

### 2. Use cursor pagination for large datasets

```python
# For feeds, timelines, logs
@app.route('/events')
def get_events():
    cursor = request.args.get('cursor')
    # Use cursor, not offset
```

### 3. Document available filters and sorts

```markdown
## Filtering

| Parameter | Type | Description |
|-----------|------|-------------|
| status | string | Filter by status (active, inactive, pending) |
| created_after | datetime | Filter by creation date |
| category | string | Filter by category (can be comma-separated) |

## Sorting

| Field | Description |
|-------|-------------|
| created_at | Sort by creation date |
| name | Sort by name |
| price | Sort by price |

Prefix with `-` for descending order.
```

### 4. Return metadata

```json
{
  "data": [...],
  "meta": {
    "total": 1000,
    "filtered": 150,
    "returned": 20
  }
}
```

### 5. Handle empty results gracefully

```json
{
  "data": [],
  "pagination": {
    "total": 0,
    "page": 1,
    "limit": 20,
    "pages": 0
  }
}
```

### 6. Validate and sanitize inputs

```python
def validate_pagination(page, limit):
    if page < 1:
        raise BadRequest("Page must be >= 1")
    if limit < 1 or limit > 100:
        raise BadRequest("Limit must be between 1 and 100")
```

---

## Key takeaways

1. **Always paginate.** Never return unbounded collections.

2. **Use cursor pagination** for large datasets and real-time feeds.

3. **Use offset pagination** for small datasets and admin UIs.

4. **Index your filter and sort columns.** Performance matters.

5. **Set sensible defaults and limits.** Don't let clients request 1 million records.

6. **Document everything.** Available filters, sorts, and their syntax.

7. **Return metadata.** Total count, current page, has_more, etc.

---

[Next: API Documentation →](./11-api-documentation.md)