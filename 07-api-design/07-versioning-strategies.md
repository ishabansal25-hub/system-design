# Versioning Strategies

[← Back to API Design](./README.md) | [← Previous: Status Codes & Error Handling](./06-status-codes-and-errors.md)

---

You will need to change your API. It's not a matter of if, but when. New features, bug fixes, performance improvements, security patches - change is inevitable.

The question is: how do you change your API without breaking every application that depends on it?

That's what versioning is for.

---

## Why versioning matters

### The problem

Imagine you have an API that returns user data:

```json
GET /users/123

{
  "id": "123",
  "name": "John Doe",
  "email": "john@example.com"
}
```

Now you want to split `name` into `first_name` and `last_name`. If you just change it:

```json
GET /users/123

{
  "id": "123",
  "first_name": "John",
  "last_name": "Doe",
  "email": "john@example.com"
}
```

Every client that expects `name` will break. Mobile apps in the App Store, third-party integrations, internal services - all broken.

### The solution

Versioning lets you make changes while keeping old clients working:

```json
GET /v1/users/123  → {"name": "John Doe", ...}
GET /v2/users/123  → {"first_name": "John", "last_name": "Doe", ...}
```

Old clients keep using v1. New clients use v2. Everyone's happy.

---

## Breaking vs. non-breaking changes

Not all changes require a new version.

### Non-breaking changes (safe)

These can be made without a new version:

| Change | Why it's safe |
|--------|---------------|
| Adding a new field | Clients should ignore unknown fields |
| Adding a new endpoint | Doesn't affect existing endpoints |
| Adding optional parameters | Existing requests still work |
| Adding new enum values | Clients should handle unknown values |
| Improving performance | Same behavior, just faster |
| Fixing bugs | Assuming the bug wasn't relied upon |

```json
// Before
{"id": "123", "name": "John"}

// After (added email - non-breaking)
{"id": "123", "name": "John", "email": "john@example.com"}
```

### Breaking changes (require new version)

These require a new version:

| Change | Why it breaks |
|--------|---------------|
| Removing a field | Clients expect it |
| Renaming a field | Clients use the old name |
| Changing a field's type | Clients parse it differently |
| Changing a field's meaning | Clients interpret it differently |
| Removing an endpoint | Clients call it |
| Changing URL structure | Clients use old URLs |
| Making optional field required | Old requests don't include it |
| Changing authentication | Old credentials don't work |
| Changing error format | Clients parse errors differently |

```json
// Before
{"id": "123", "name": "John"}

// After (renamed name to full_name - BREAKING)
{"id": "123", "full_name": "John"}
```

### Gray areas

Some changes are debatable:

**Changing default values:**
```json
// Before: limit defaults to 10
GET /users → returns 10 users

// After: limit defaults to 20
GET /users → returns 20 users
```

Technically non-breaking, but might surprise clients.

**Adding required fields to responses:**
```json
// Before
{"id": "123", "name": "John"}

// After (added required email)
{"id": "123", "name": "John", "email": "john@example.com"}
```

Non-breaking for most clients, but might break strict schema validators.

**My rule:** If in doubt, treat it as breaking.

---

## Versioning strategies

There are several ways to version an API. Each has tradeoffs.

### 1. URL path versioning

The version is part of the URL path.

```
GET /v1/users/123
GET /v2/users/123
```

**Pros:**
- Very explicit and visible
- Easy to understand
- Easy to route at infrastructure level
- Easy to test in browser
- Clear which version you're using

**Cons:**
- Not "pure" REST (version isn't part of the resource)
- Can lead to URL proliferation
- Harder to share URLs (which version?)

**Implementation:**
```
/v1/users/123
/v2/users/123
/api/v1/users/123
```

**This is my recommendation for most APIs.** It's the most widely used and understood approach.

### 2. Query parameter versioning

The version is a query parameter.

```
GET /users/123?version=1
GET /users/123?api-version=2
GET /users/123?v=1
```

**Pros:**
- Clean URLs
- Easy to default to latest version
- Can be optional

**Cons:**
- Easy to forget
- Caching can be tricky (same URL, different responses)
- Less visible than URL path

**Implementation:**
```
/users/123?version=1
/users/123?api-version=2024-01-15
```

### 3. Header versioning

The version is in a custom header.

```http
GET /users/123 HTTP/1.1
X-API-Version: 1

GET /users/123 HTTP/1.1
X-API-Version: 2
```

**Pros:**
- Clean URLs
- "Pure" REST (URL identifies resource)
- Can be optional

**Cons:**
- Not visible in URL
- Harder to test in browser
- Easy to forget
- Requires documentation

**Implementation:**
```http
X-API-Version: 1
X-API-Version: 2024-01-15
API-Version: 1
```

### 4. Accept header versioning (content negotiation)

The version is in the Accept header using vendor media types.

```http
GET /users/123 HTTP/1.1
Accept: application/vnd.example.v1+json

GET /users/123 HTTP/1.1
Accept: application/vnd.example.v2+json
```

**Pros:**
- "Pure" REST
- Uses standard HTTP content negotiation
- Can version individual resources differently

**Cons:**
- Complex
- Not visible in URL
- Harder to test
- Less widely understood

**Implementation:**
```http
Accept: application/vnd.myapi.v1+json
Accept: application/vnd.myapi+json; version=1
Accept: application/json; version=1
```

### 5. Date-based versioning

The version is a date, representing when the API behavior was frozen.

```
GET /2024-01-15/users/123
GET /users/123?api-version=2024-01-15
```

**Pros:**
- Clear when behavior was introduced
- No arbitrary version numbers
- Good for APIs that change frequently

**Cons:**
- Harder to remember dates
- Not clear what changed between dates
- Can have many versions

**Used by:** Stripe, Azure

**Implementation:**
```
/2024-01-15/users/123
X-API-Version: 2024-01-15
?api-version=2024-01-15
```

### Comparison

| Strategy | Visibility | Simplicity | REST Purity | Caching |
|----------|------------|------------|-------------|---------|
| URL path | High | High | Low | Easy |
| Query param | Medium | High | Medium | Tricky |
| Header | Low | Medium | High | Easy |
| Accept header | Low | Low | High | Easy |
| Date-based | Medium | Medium | Varies | Varies |

---

## Version lifecycle

### Version states

| State | Description |
|-------|-------------|
| **Current** | The recommended version for new integrations |
| **Supported** | Still maintained, but not recommended for new integrations |
| **Deprecated** | Will be removed, clients should migrate |
| **Sunset** | No longer available |

### Deprecation process

1. **Announce deprecation** - Give plenty of notice (6-12 months minimum)
2. **Add deprecation headers** - Warn clients programmatically
3. **Monitor usage** - Track who's still using deprecated versions
4. **Reach out** - Contact heavy users directly
5. **Sunset** - Remove the version

### Deprecation headers

```http
HTTP/1.1 200 OK
Deprecation: true
Sunset: Sat, 15 Jun 2025 00:00:00 GMT
Link: <https://api.example.com/docs/migration>; rel="deprecation"

{
  "id": "123",
  "name": "John Doe"
}
```

Or in the response body:

```json
{
  "data": {...},
  "meta": {
    "deprecation_warning": "This API version is deprecated and will be removed on 2025-06-15. Please migrate to v2.",
    "migration_guide": "https://api.example.com/docs/v1-to-v2-migration"
  }
}
```

### Sunset policy

Document your sunset policy clearly:

```
Version Lifecycle Policy:
- New versions are supported for at least 2 years
- Deprecated versions are maintained for at least 1 year after deprecation
- We provide at least 6 months notice before sunsetting a version
- Critical security fixes are backported to all supported versions
```

---

## Managing multiple versions

### Code organization

**Option 1: Separate codebases**
```
/api-v1/
  /src/
  /tests/
/api-v2/
  /src/
  /tests/
```

Pros: Complete isolation
Cons: Duplication, hard to maintain

**Option 2: Version folders**
```
/src/
  /v1/
    /controllers/
    /models/
  /v2/
    /controllers/
    /models/
  /shared/
```

Pros: Some code sharing
Cons: Can get messy

**Option 3: Version adapters**
```
/src/
  /controllers/
  /models/
  /adapters/
    /v1/
    /v2/
```

Pros: Single source of truth, adapters handle differences
Cons: Adapter complexity

**My preference:** Option 3 for most cases. Keep business logic in one place, use adapters to transform requests/responses for different versions.

### Routing

```python
# URL path versioning
@app.route('/v1/users/<id>')
def get_user_v1(id):
    user = get_user(id)
    return v1_adapter.transform(user)

@app.route('/v2/users/<id>')
def get_user_v2(id):
    user = get_user(id)
    return v2_adapter.transform(user)
```

```python
# Header versioning
@app.route('/users/<id>')
def get_user(id):
    version = request.headers.get('X-API-Version', '2')
    user = get_user(id)
    
    if version == '1':
        return v1_adapter.transform(user)
    else:
        return v2_adapter.transform(user)
```

### Database considerations

When your data model changes:

**Option 1: Support both formats in DB**
```sql
-- Store both old and new format
ALTER TABLE users ADD COLUMN first_name VARCHAR(100);
ALTER TABLE users ADD COLUMN last_name VARCHAR(100);
-- Keep 'name' column for v1 compatibility
```

**Option 2: Compute old format on read**
```python
def get_user_v1(id):
    user = db.get_user(id)
    return {
        "id": user.id,
        "name": f"{user.first_name} {user.last_name}",  # Computed
        "email": user.email
    }
```

**Option 3: Store canonical format, transform on read/write**
```python
# DB stores first_name, last_name
# v1 adapter combines them for response
# v1 adapter splits them for requests
```

---

## Versioning best practices

### 1. Version from day one

Don't wait until you need to make a breaking change.

```
Bad:  /users/123 (no version)
Good: /v1/users/123 (versioned from start)
```

### 2. Keep versions to a minimum

Every version is maintenance burden. Aim for 2-3 active versions max.

```
Ideal:
- v1 (deprecated)
- v2 (current)
- v3 (beta)
```

### 3. Make non-breaking changes when possible

Before creating a new version, ask: can I make this change non-breaking?

```
Instead of:
v1: {"name": "John Doe"}
v2: {"first_name": "John", "last_name": "Doe"}

Consider:
v1: {"name": "John Doe", "first_name": "John", "last_name": "Doe"}
(Add new fields, keep old ones)
```

### 4. Document changes clearly

Maintain a changelog:

```markdown
## v2.0.0 (2024-01-15)

### Breaking Changes
- `name` field split into `first_name` and `last_name`
- `created_at` now returns ISO 8601 format instead of Unix timestamp

### New Features
- Added `phone` field to user resource
- Added `GET /users/{id}/orders` endpoint

### Deprecations
- `name` field is deprecated, use `first_name` and `last_name`
```

### 5. Provide migration guides

Help developers upgrade:

```markdown
# Migrating from v1 to v2

## User resource changes

### Name field
v1: `{"name": "John Doe"}`
v2: `{"first_name": "John", "last_name": "Doe"}`

To migrate:
1. Update your code to use `first_name` and `last_name`
2. If you need the full name, concatenate: `${first_name} ${last_name}`

### Code example
```javascript
// Before (v1)
const fullName = user.name;

// After (v2)
const fullName = `${user.first_name} ${user.last_name}`;
```
```

### 6. Use feature flags for gradual rollout

```python
def get_user(id, version):
    user = db.get_user(id)
    
    # Gradual rollout of v2 features
    if version == '2' or feature_flags.is_enabled('v2_user_format', user.id):
        return v2_format(user)
    else:
        return v1_format(user)
```

### 7. Monitor version usage

Track which versions are being used:

```
Version Usage (last 30 days):
- v1: 15% of requests (deprecated)
- v2: 80% of requests (current)
- v3: 5% of requests (beta)

Top v1 users:
- client_abc: 1.2M requests
- client_xyz: 800K requests
```

---

## Real-world examples

### Stripe (date-based)

```http
GET /v1/customers/cus_123 HTTP/1.1
Stripe-Version: 2024-01-15
```

- Uses date-based versioning
- Version in header
- Accounts can be pinned to a version
- Detailed changelog for each version

### GitHub (Accept header)

```http
GET /users/octocat HTTP/1.1
Accept: application/vnd.github.v3+json
```

- Uses Accept header versioning
- Also supports URL versioning for some endpoints
- Provides preview features via Accept header

### Twilio (URL path with date)

```
GET /2010-04-01/Accounts/AC123/Messages
```

- Date in URL path
- Very stable API (same date for years)
- Clear when API was introduced

### Twitter/X (URL path)

```
GET /2/users/123
GET /2/tweets/456
```

- Simple numeric versioning
- Version in URL path
- Major versions only

---

## Common mistakes

### 1. Not versioning at all

```
Bad:  /users/123 (no version, can't evolve)
Good: /v1/users/123
```

### 2. Too many versions

```
Bad:  v1, v1.1, v1.2, v2, v2.1, v2.2, v3... (maintenance nightmare)
Good: v1, v2, v3 (major versions only)
```

### 3. Breaking changes without new version

```
Bad:  Changing field names in v1 (breaks clients)
Good: Create v2 with new field names
```

### 4. No deprecation period

```
Bad:  "v1 is removed effective immediately"
Good: "v1 is deprecated, will be removed in 12 months"
```

### 5. Inconsistent versioning

```
Bad:  /v1/users, /api/2/orders, /products?version=3
Good: /v1/users, /v1/orders, /v1/products
```

### 6. Versioning too granularly

```
Bad:  Different versions for different endpoints
Good: One version for the entire API
```

---

## Key takeaways

1. **Version from day one.** Don't wait until you need it.

2. **Use URL path versioning.** It's the most explicit and widely understood.

3. **Minimize breaking changes.** Add fields instead of changing them when possible.

4. **Keep versions to a minimum.** 2-3 active versions is ideal.

5. **Deprecate gracefully.** Give plenty of notice and provide migration guides.

6. **Document everything.** Changelogs, migration guides, sunset dates.

7. **Monitor usage.** Know who's using which version.

---

[Next: Authentication & Authorization →](./08-authentication-authorization.md)