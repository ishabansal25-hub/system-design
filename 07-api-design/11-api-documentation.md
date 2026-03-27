# API Documentation

[← Back to API Design](./README.md) | [← Previous: Pagination, Filtering & Sorting](./10-pagination-filtering-sorting.md)

---

Documentation is where most APIs fail. You can have the most elegant, well-designed API in the world, but if developers can't figure out how to use it, they'll find another one.

I've abandoned APIs because the documentation was so bad I couldn't make a single successful request. Don't let that be your API.

---

## Why documentation matters

```
Good documentation:
- Developers integrate in hours, not days
- Fewer support tickets
- Higher adoption rates
- Happy developers who recommend your API

Bad documentation:
- Developers give up and use competitors
- Support team overwhelmed with basic questions
- Low adoption despite good API design
- Frustrated developers who warn others away
```

---

## Documentation components

### 1. Getting started guide

The first thing developers see. Get them to a successful API call in 5 minutes.

```markdown
# Getting Started

## 1. Get your API key

Sign up at https://api.example.com/signup to get your API key.

## 2. Make your first request

```bash
curl -X GET "https://api.example.com/v1/users/me" \
  -H "Authorization: Bearer YOUR_API_KEY"
```

## 3. Explore the API

Now that you've made your first request, explore:
- [Users API](/docs/users) - Manage user accounts
- [Orders API](/docs/orders) - Create and track orders
- [Products API](/docs/products) - Browse product catalog
```

### 2. Authentication guide

Clear instructions on how to authenticate.

```markdown
# Authentication

All API requests require authentication using an API key.

## Getting your API key

1. Log in to your dashboard
2. Navigate to Settings → API Keys
3. Click "Create New Key"
4. Copy your key (it won't be shown again)

## Using your API key

Include your API key in the `Authorization` header:

```bash
curl -X GET "https://api.example.com/v1/users" \
  -H "Authorization: Bearer sk_live_abc123"
```

## Test vs. Live keys

| Key prefix | Environment | Real data? |
|------------|-------------|------------|
| sk_test_   | Sandbox     | No         |
| sk_live_   | Production  | Yes        |

Use test keys during development. Switch to live keys for production.
```

### 3. API reference

Detailed documentation for every endpoint.

```markdown
# Create User

Creates a new user account.

## Endpoint

```
POST /v1/users
```

## Request

### Headers

| Header | Required | Description |
|--------|----------|-------------|
| Authorization | Yes | Bearer token |
| Content-Type | Yes | application/json |

### Body Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| email | string | Yes | User's email address |
| name | string | Yes | User's full name |
| password | string | Yes | Password (min 8 characters) |
| role | string | No | User role (default: "user") |

### Example Request

```bash
curl -X POST "https://api.example.com/v1/users" \
  -H "Authorization: Bearer sk_live_abc123" \
  -H "Content-Type: application/json" \
  -d '{
    "email": "john@example.com",
    "name": "John Doe",
    "password": "securepassword123"
  }'
```

## Response

### Success (201 Created)

```json
{
  "id": "user_abc123",
  "email": "john@example.com",
  "name": "John Doe",
  "role": "user",
  "created_at": "2024-01-15T10:30:00Z"
}
```

### Errors

| Status | Code | Description |
|--------|------|-------------|
| 400 | INVALID_REQUEST | Request body is malformed |
| 409 | EMAIL_EXISTS | Email already registered |
| 422 | VALIDATION_ERROR | Validation failed |

### Example Error

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Validation failed",
    "details": [
      {"field": "email", "message": "Invalid email format"},
      {"field": "password", "message": "Must be at least 8 characters"}
    ]
  }
}
```
```

### 4. Code examples

Examples in multiple languages.

```markdown
# Code Examples

## Create a user

### cURL

```bash
curl -X POST "https://api.example.com/v1/users" \
  -H "Authorization: Bearer sk_live_abc123" \
  -H "Content-Type: application/json" \
  -d '{"email": "john@example.com", "name": "John Doe"}'
```

### Python

```python
import requests

response = requests.post(
    "https://api.example.com/v1/users",
    headers={
        "Authorization": "Bearer sk_live_abc123",
        "Content-Type": "application/json"
    },
    json={
        "email": "john@example.com",
        "name": "John Doe"
    }
)

user = response.json()
print(f"Created user: {user['id']}")
```

### JavaScript

```javascript
const response = await fetch("https://api.example.com/v1/users", {
  method: "POST",
  headers: {
    "Authorization": "Bearer sk_live_abc123",
    "Content-Type": "application/json"
  },
  body: JSON.stringify({
    email: "john@example.com",
    name: "John Doe"
  })
});

const user = await response.json();
console.log(`Created user: ${user.id}`);
```

### Ruby

```ruby
require 'net/http'
require 'json'

uri = URI("https://api.example.com/v1/users")
http = Net::HTTP.new(uri.host, uri.port)
http.use_ssl = true

request = Net::HTTP::Post.new(uri)
request["Authorization"] = "Bearer sk_live_abc123"
request["Content-Type"] = "application/json"
request.body = { email: "john@example.com", name: "John Doe" }.to_json

response = http.request(request)
user = JSON.parse(response.body)
puts "Created user: #{user['id']}"
```
```

### 5. Error reference

Document all possible errors.

```markdown
# Errors

## Error format

All errors follow this format:

```json
{
  "error": {
    "code": "ERROR_CODE",
    "message": "Human-readable message",
    "details": [...],
    "request_id": "req_abc123"
  }
}
```

## Error codes

### Authentication errors

| Code | Status | Description |
|------|--------|-------------|
| UNAUTHORIZED | 401 | Missing or invalid API key |
| INVALID_TOKEN | 401 | Token is expired or malformed |
| FORBIDDEN | 403 | Valid token but insufficient permissions |

### Validation errors

| Code | Status | Description |
|------|--------|-------------|
| INVALID_REQUEST | 400 | Request body is malformed |
| VALIDATION_ERROR | 422 | One or more fields failed validation |
| MISSING_FIELD | 422 | Required field is missing |

### Resource errors

| Code | Status | Description |
|------|--------|-------------|
| NOT_FOUND | 404 | Resource doesn't exist |
| CONFLICT | 409 | Resource already exists |
| GONE | 410 | Resource was deleted |

### Rate limiting

| Code | Status | Description |
|------|--------|-------------|
| RATE_LIMITED | 429 | Too many requests |

### Server errors

| Code | Status | Description |
|------|--------|-------------|
| INTERNAL_ERROR | 500 | Unexpected server error |
| SERVICE_UNAVAILABLE | 503 | Service temporarily unavailable |
```

### 6. Changelog

Keep developers informed of changes.

```markdown
# Changelog

## 2024-01-15

### Added
- New `phone` field on User resource
- `GET /v1/users/{id}/orders` endpoint

### Changed
- `created_at` now returns ISO 8601 format (was Unix timestamp)

### Deprecated
- `name` field on User (use `first_name` and `last_name`)

### Fixed
- Fixed pagination returning duplicate results

## 2024-01-01

### Added
- Initial API release
```

---

## OpenAPI (Swagger)

OpenAPI is the standard for describing REST APIs.

### Basic structure

```yaml
openapi: 3.0.3
info:
  title: Example API
  description: API for managing users and orders
  version: 1.0.0
  contact:
    email: api@example.com
  license:
    name: MIT

servers:
  - url: https://api.example.com/v1
    description: Production
  - url: https://sandbox.api.example.com/v1
    description: Sandbox

paths:
  /users:
    get:
      summary: List users
      operationId: listUsers
      tags:
        - Users
      parameters:
        - name: limit
          in: query
          schema:
            type: integer
            default: 20
            maximum: 100
        - name: offset
          in: query
          schema:
            type: integer
            default: 0
      responses:
        '200':
          description: List of users
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/UserList'
    
    post:
      summary: Create user
      operationId: createUser
      tags:
        - Users
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/CreateUserRequest'
      responses:
        '201':
          description: User created
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/User'
        '422':
          description: Validation error
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Error'

components:
  schemas:
    User:
      type: object
      properties:
        id:
          type: string
          example: user_abc123
        email:
          type: string
          format: email
          example: john@example.com
        name:
          type: string
          example: John Doe
        created_at:
          type: string
          format: date-time
      required:
        - id
        - email
        - name
        - created_at
    
    CreateUserRequest:
      type: object
      properties:
        email:
          type: string
          format: email
        name:
          type: string
          minLength: 1
          maxLength: 100
        password:
          type: string
          minLength: 8
      required:
        - email
        - name
        - password
    
    Error:
      type: object
      properties:
        error:
          type: object
          properties:
            code:
              type: string
            message:
              type: string
            details:
              type: array
              items:
                type: object

  securitySchemes:
    bearerAuth:
      type: http
      scheme: bearer

security:
  - bearerAuth: []
```

### Benefits of OpenAPI

1. **Auto-generate documentation** - Swagger UI, ReDoc
2. **Auto-generate client SDKs** - OpenAPI Generator
3. **Validate requests/responses** - Ensure API matches spec
4. **Import into tools** - Postman, Insomnia

---

## Documentation tools

### Swagger UI

Interactive documentation from OpenAPI spec.

```html
<!DOCTYPE html>
<html>
<head>
  <title>API Documentation</title>
  <link rel="stylesheet" href="https://unpkg.com/swagger-ui-dist/swagger-ui.css">
</head>
<body>
  <div id="swagger-ui"></div>
  <script src="https://unpkg.com/swagger-ui-dist/swagger-ui-bundle.js"></script>
  <script>
    SwaggerUIBundle({
      url: "/openapi.yaml",
      dom_id: '#swagger-ui'
    });
  </script>
</body>
</html>
```

### ReDoc

Beautiful, responsive documentation.

```html
<!DOCTYPE html>
<html>
<head>
  <title>API Documentation</title>
</head>
<body>
  <redoc spec-url="/openapi.yaml"></redoc>
  <script src="https://cdn.redoc.ly/redoc/latest/bundles/redoc.standalone.js"></script>
</body>
</html>
```

### Postman

- Import OpenAPI spec
- Create collections
- Share with team
- Generate documentation

### ReadMe.io

- Hosted documentation
- API explorer
- Metrics and analytics
- Versioning

---

## Interactive examples

### Try it now

Let developers make real API calls from documentation.

```markdown
# Try it now

<api-explorer>
  <endpoint method="GET" path="/v1/users">
    <parameter name="limit" type="integer" default="20" />
    <parameter name="status" type="string" enum="active,inactive" />
  </endpoint>
</api-explorer>
```

### Sandbox environment

Provide a safe environment for testing.

```markdown
# Sandbox

Use our sandbox environment for testing:

- Base URL: `https://sandbox.api.example.com/v1`
- Test API key: `sk_test_demo123`
- Data resets daily at midnight UTC

## Test data

The sandbox includes pre-populated test data:

| Resource | ID | Description |
|----------|-----|-------------|
| User | user_test_1 | Test user account |
| Product | prod_test_1 | Sample product |
| Order | order_test_1 | Sample order |
```

---

## SDKs and client libraries

### Official SDKs

```markdown
# SDKs

We provide official SDKs for popular languages:

## Python

```bash
pip install example-api
```

```python
from example_api import Client

client = Client(api_key="sk_live_abc123")
user = client.users.create(
    email="john@example.com",
    name="John Doe"
)
```

## JavaScript/Node.js

```bash
npm install @example/api
```

```javascript
import { Client } from '@example/api';

const client = new Client({ apiKey: 'sk_live_abc123' });
const user = await client.users.create({
  email: 'john@example.com',
  name: 'John Doe'
});
```

## Ruby

```bash
gem install example-api
```

```ruby
require 'example_api'

client = ExampleAPI::Client.new(api_key: 'sk_live_abc123')
user = client.users.create(
  email: 'john@example.com',
  name: 'John Doe'
)
```
```

---

## Best practices

### 1. Start with getting started

```markdown
# Getting Started

Get your first API call working in 5 minutes.

1. Sign up for an account
2. Get your API key
3. Make your first request
4. Explore the API
```

### 2. Show, don't tell

```markdown
# Bad
The API uses JSON for request and response bodies.

# Good
Request:
```json
{"email": "john@example.com", "name": "John Doe"}
```

Response:
```json
{"id": "user_123", "email": "john@example.com", "name": "John Doe"}
```
```

### 3. Provide copy-paste examples

```bash
# This should work when pasted into terminal
curl -X GET "https://api.example.com/v1/users" \
  -H "Authorization: Bearer YOUR_API_KEY"
```

### 4. Document errors thoroughly

```markdown
## Common errors

### Invalid API key

```json
{
  "error": {
    "code": "UNAUTHORIZED",
    "message": "Invalid API key"
  }
}
```

**Solution:** Check that your API key is correct and hasn't expired.
```

### 5. Keep it up to date

- Update docs with every API change
- Mark deprecated features clearly
- Include version in documentation

### 6. Make it searchable

- Good navigation
- Search functionality
- Clear headings

### 7. Include rate limits and quotas

```markdown
## Rate Limits

| Tier | Requests/minute | Requests/day |
|------|-----------------|--------------|
| Free | 60 | 1,000 |
| Pro | 600 | 50,000 |

Rate limit headers are included in every response.
```

---

## Documentation checklist

Before launching your API, ensure you have:

- [ ] Getting started guide (< 5 minutes to first call)
- [ ] Authentication documentation
- [ ] Complete API reference for all endpoints
- [ ] Request/response examples for every endpoint
- [ ] Error documentation with solutions
- [ ] Code examples in multiple languages
- [ ] Rate limiting documentation
- [ ] Changelog
- [ ] SDKs or client libraries
- [ ] Sandbox/test environment
- [ ] Search functionality
- [ ] Mobile-friendly design

---

## Key takeaways

1. **Documentation is part of your API.** Treat it with the same care as your code.

2. **Get developers to success quickly.** 5 minutes to first successful call.

3. **Show, don't tell.** Examples are worth a thousand words.

4. **Use OpenAPI.** It's the standard, and tools can generate docs from it.

5. **Keep it updated.** Outdated docs are worse than no docs.

6. **Make it interactive.** Let developers try the API from the docs.

7. **Provide SDKs.** Make integration as easy as possible.

---

[Next: GraphQL Essentials →](./12-graphql-essentials.md)