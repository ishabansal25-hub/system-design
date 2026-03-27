# Authentication & Authorization

[← Back to API Design](./README.md) | [← Previous: Versioning Strategies](./07-versioning-strategies.md)

---

Authentication and authorization are often confused, but they're fundamentally different:

- **Authentication (AuthN):** "Who are you?" - Verifying identity
- **Authorization (AuthZ):** "What can you do?" - Verifying permissions

Get these wrong, and you'll have security breaches, data leaks, and very unhappy users. Get them right, and they become invisible - which is exactly what you want.

---

## Authentication methods

### 1. API Keys

The simplest form of authentication. A unique string that identifies the client.

```http
GET /users HTTP/1.1
Host: api.example.com
X-API-Key: sk_live_abc123xyz789
```

Or in query parameter (less secure):
```http
GET /users?api_key=sk_live_abc123xyz789
```

**How it works:**
1. Client registers and receives an API key
2. Client includes key in every request
3. Server validates key and identifies the client

**Pros:**
- Simple to implement
- Easy for developers to use
- Good for server-to-server communication

**Cons:**
- No expiration (unless you build it)
- If leaked, attacker has permanent access
- No user context (identifies app, not user)
- Can't be used in browsers (visible in network tab)

**Best practices:**
```
✓ Use different keys for test/production
✓ Prefix keys to identify type: sk_live_, sk_test_
✓ Allow key rotation without downtime
✓ Rate limit per key
✓ Log key usage for auditing
✗ Don't put keys in URLs (they get logged)
✗ Don't embed keys in client-side code
```

**When to use:** Server-to-server APIs, internal services, simple integrations.

---

### 2. Basic Authentication

Username and password encoded in Base64.

```http
GET /users HTTP/1.1
Host: api.example.com
Authorization: Basic am9objpzZWNyZXQxMjM=
```

The value is `base64(username:password)`:
```
base64("john:secret123") = "am9objpzZWNyZXQxMjM="
```

**How it works:**
1. Client encodes credentials as `base64(username:password)`
2. Client sends in `Authorization` header
3. Server decodes and validates credentials

**Pros:**
- Simple and widely supported
- Built into HTTP standard
- Works with any HTTP client

**Cons:**
- Credentials sent with every request
- Base64 is encoding, not encryption (easily decoded)
- Must use HTTPS (credentials visible otherwise)
- No session management

**When to use:** Simple APIs, internal tools, when combined with HTTPS. Generally avoid for public APIs.

---

### 3. Bearer Tokens (JWT)

A token that proves the client has been authenticated.

```http
GET /users HTTP/1.1
Host: api.example.com
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

#### JWT (JSON Web Token) Structure

A JWT has three parts separated by dots:

```
header.payload.signature

eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.
eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4iLCJpYXQiOjE1MTYyMzkwMjJ9.
SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c
```

**Header:**
```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```

**Payload (claims):**
```json
{
  "sub": "user_123",           // Subject (user ID)
  "name": "John Doe",          // Custom claim
  "email": "john@example.com", // Custom claim
  "role": "admin",             // Custom claim
  "iat": 1516239022,           // Issued at
  "exp": 1516242622,           // Expiration
  "iss": "api.example.com"     // Issuer
}
```

**Signature:**
```
HMACSHA256(
  base64UrlEncode(header) + "." + base64UrlEncode(payload),
  secret
)
```

**How it works:**
1. Client authenticates (username/password, OAuth, etc.)
2. Server creates JWT with user info and signs it
3. Client stores JWT and sends with every request
4. Server validates signature and extracts user info

**Pros:**
- Stateless (no server-side session storage)
- Contains user info (no database lookup needed)
- Can be verified without contacting auth server
- Works across services (microservices)

**Cons:**
- Can't be invalidated before expiration (without blacklist)
- Payload is readable (don't put secrets in it)
- Can get large with many claims
- Token theft = account compromise until expiration

**Best practices:**
```
✓ Keep tokens short-lived (15 min - 1 hour)
✓ Use refresh tokens for long sessions
✓ Store securely (httpOnly cookies or secure storage)
✓ Validate signature, expiration, and issuer
✓ Use strong secrets (256+ bits)
✗ Don't store sensitive data in payload
✗ Don't use JWT for sessions if you need revocation
```

---

### 4. OAuth 2.0

An authorization framework that lets users grant limited access to their resources without sharing credentials.

**The players:**
- **Resource Owner:** The user who owns the data
- **Client:** The app wanting access
- **Authorization Server:** Issues tokens (e.g., Google, GitHub)
- **Resource Server:** Hosts the protected resources (your API)

#### OAuth 2.0 Flows

**Authorization Code Flow (most secure, for server-side apps)**

```
1. User clicks "Login with Google"
2. App redirects to Google:
   GET https://accounts.google.com/oauth/authorize?
     response_type=code&
     client_id=YOUR_CLIENT_ID&
     redirect_uri=https://yourapp.com/callback&
     scope=email profile&
     state=random_string

3. User logs in and grants permission
4. Google redirects back with code:
   GET https://yourapp.com/callback?code=AUTH_CODE&state=random_string

5. App exchanges code for tokens (server-side):
   POST https://oauth2.googleapis.com/token
   {
     "grant_type": "authorization_code",
     "code": "AUTH_CODE",
     "client_id": "YOUR_CLIENT_ID",
     "client_secret": "YOUR_CLIENT_SECRET",
     "redirect_uri": "https://yourapp.com/callback"
   }

6. Google returns tokens:
   {
     "access_token": "ya29.xxx",
     "refresh_token": "1//xxx",
     "expires_in": 3600,
     "token_type": "Bearer"
   }

7. App uses access_token to call APIs
```

**PKCE Flow (for mobile/SPA apps)**

Same as Authorization Code, but uses a code verifier instead of client secret:

```
1. Generate code_verifier (random string)
2. Generate code_challenge = SHA256(code_verifier)
3. Include code_challenge in authorization request
4. Include code_verifier when exchanging code for token
```

**Client Credentials Flow (for machine-to-machine)**

```
POST https://oauth2.example.com/token
{
  "grant_type": "client_credentials",
  "client_id": "YOUR_CLIENT_ID",
  "client_secret": "YOUR_CLIENT_SECRET",
  "scope": "read:users"
}
```

No user involved - the app authenticates as itself.

**When to use which flow:**

| Flow | Use case |
|------|----------|
| Authorization Code | Server-side web apps |
| Authorization Code + PKCE | Mobile apps, SPAs |
| Client Credentials | Server-to-server, microservices |
| Implicit (deprecated) | Don't use |
| Password (deprecated) | Don't use |

---

### 5. Session-based Authentication

Traditional approach using server-side sessions.

```http
POST /login HTTP/1.1
Content-Type: application/json

{"username": "john", "password": "secret"}

HTTP/1.1 200 OK
Set-Cookie: session_id=abc123; HttpOnly; Secure; SameSite=Strict

---

GET /users HTTP/1.1
Cookie: session_id=abc123
```

**How it works:**
1. Client sends credentials
2. Server creates session, stores in database/cache
3. Server sends session ID in cookie
4. Client sends cookie with every request
5. Server looks up session to identify user

**Pros:**
- Easy to invalidate (delete session)
- Session data stored server-side (secure)
- Works well with browsers

**Cons:**
- Requires server-side storage
- Harder to scale (sticky sessions or shared storage)
- Doesn't work well for APIs consumed by non-browsers

**When to use:** Traditional web applications, when you need easy session invalidation.

---

## Token management

### Access tokens vs. Refresh tokens

| Aspect | Access Token | Refresh Token |
|--------|--------------|---------------|
| Purpose | Access resources | Get new access tokens |
| Lifetime | Short (15 min - 1 hour) | Long (days - months) |
| Storage | Memory or secure storage | Secure storage only |
| Sent to | Resource server | Auth server only |

**The flow:**
```
1. User logs in
2. Server returns access_token (15 min) + refresh_token (30 days)
3. Client uses access_token for API calls
4. Access token expires
5. Client uses refresh_token to get new access_token
6. Repeat until refresh_token expires
7. User must log in again
```

**Token refresh endpoint:**
```http
POST /oauth/token HTTP/1.1
Content-Type: application/json

{
  "grant_type": "refresh_token",
  "refresh_token": "dGhpcyBpcyBhIHJlZnJlc2ggdG9rZW4..."
}

HTTP/1.1 200 OK
{
  "access_token": "new_access_token...",
  "expires_in": 900,
  "refresh_token": "new_refresh_token..."  // Optional: rotate refresh token
}
```

### Token storage

| Storage | Security | Use case |
|---------|----------|----------|
| Memory | High | SPAs (lost on refresh) |
| HttpOnly Cookie | High | Web apps |
| localStorage | Low | Avoid for tokens |
| sessionStorage | Medium | Short-lived tokens |
| Secure Enclave | High | Mobile apps |

**Never store tokens in:**
- localStorage (XSS vulnerable)
- URL parameters (logged, cached)
- Plain cookies (CSRF vulnerable)

### Token revocation

For JWTs, you need a revocation strategy:

**Option 1: Short expiration**
```
Access tokens expire in 15 minutes.
If compromised, damage is limited.
```

**Option 2: Token blacklist**
```python
# On logout or security event
blacklist.add(token_id, expiration_time)

# On every request
if token_id in blacklist:
    return 401 Unauthorized
```

**Option 3: Token versioning**
```python
# User has a token_version in database
# JWT includes token_version claim
# On logout, increment user's token_version
# All old tokens become invalid
```

---

## Authorization patterns

### Role-Based Access Control (RBAC)

Users are assigned roles, roles have permissions.

```
Roles:
- admin: can do everything
- editor: can read, create, update
- viewer: can only read

Users:
- john: admin
- jane: editor
- bob: viewer
```

**Implementation:**
```python
# In JWT
{
  "sub": "user_123",
  "roles": ["editor"]
}

# In code
@require_role("editor")
def update_article(article_id):
    ...

def require_role(role):
    def decorator(f):
        def wrapper(*args, **kwargs):
            if role not in current_user.roles:
                raise Forbidden()
            return f(*args, **kwargs)
        return wrapper
    return decorator
```

**Pros:** Simple, easy to understand
**Cons:** Can lead to role explosion, not granular

### Permission-Based Access Control

Users have specific permissions.

```
Permissions:
- articles:read
- articles:create
- articles:update
- articles:delete
- users:read
- users:manage

Users:
- john: articles:*, users:*
- jane: articles:read, articles:create, articles:update
- bob: articles:read
```

**Implementation:**
```python
# In JWT
{
  "sub": "user_123",
  "permissions": ["articles:read", "articles:update"]
}

# In code
@require_permission("articles:update")
def update_article(article_id):
    ...
```

**Pros:** Granular control
**Cons:** More complex to manage

### Attribute-Based Access Control (ABAC)

Decisions based on attributes of user, resource, and environment.

```
Policy: "Users can edit articles they created"

Attributes:
- user.id = "123"
- article.author_id = "123"
- action = "update"

Decision: user.id == article.author_id → ALLOW
```

**Implementation:**
```python
def can_update_article(user, article):
    # User is author
    if article.author_id == user.id:
        return True
    # User is admin
    if "admin" in user.roles:
        return True
    # User is editor and article is in their department
    if "editor" in user.roles and article.department == user.department:
        return True
    return False
```

**Pros:** Very flexible, handles complex scenarios
**Cons:** Complex to implement and audit

### Resource-Based Authorization

Check permissions on specific resources.

```http
GET /articles/123 HTTP/1.1
Authorization: Bearer eyJ...

# Server checks:
# 1. Is token valid?
# 2. Does user have permission to read this specific article?
```

**Implementation:**
```python
def get_article(article_id):
    article = db.get_article(article_id)
    
    if not article:
        raise NotFound()
    
    # Check resource-level permission
    if not can_access(current_user, article, "read"):
        raise Forbidden()
    
    return article

def can_access(user, article, action):
    # Public articles can be read by anyone
    if action == "read" and article.is_public:
        return True
    # Author can do anything
    if article.author_id == user.id:
        return True
    # Check explicit permissions
    return has_permission(user, f"articles:{article.id}:{action}")
```

---

## OAuth 2.0 scopes

Scopes define what access is being requested.

```http
GET /oauth/authorize?
  scope=read:user write:articles read:articles
```

**Common scope patterns:**

```
# Resource:action
read:users
write:users
delete:users

# Or just resource (implies all actions)
users
articles

# Or hierarchical
users:profile:read
users:profile:write
users:email:read
```

**Scope validation:**
```python
@require_scope("write:articles")
def create_article():
    ...

def require_scope(scope):
    def decorator(f):
        def wrapper(*args, **kwargs):
            if scope not in current_token.scopes:
                raise Forbidden(f"Missing scope: {scope}")
            return f(*args, **kwargs)
        return wrapper
    return decorator
```

**Scope consent screen:**
```
"ArticleApp" wants to:
☑ Read your profile information (read:profile)
☑ Read your articles (read:articles)
☑ Create and edit articles (write:articles)
☐ Delete your articles (delete:articles)

[Allow] [Deny]
```

---

## Security best practices

### Token security

```
✓ Use HTTPS everywhere
✓ Set short expiration times
✓ Use secure, random secrets (256+ bits)
✓ Validate all claims (exp, iss, aud)
✓ Use HttpOnly, Secure, SameSite cookies
✓ Implement token rotation
✓ Log authentication events

✗ Don't store tokens in localStorage
✗ Don't put tokens in URLs
✗ Don't use weak secrets
✗ Don't trust client-provided data
```

### Password security

```
✓ Hash passwords (bcrypt, argon2)
✓ Use salt (built into bcrypt/argon2)
✓ Enforce minimum complexity
✓ Check against breached password lists
✓ Implement rate limiting on login
✓ Use MFA for sensitive operations

✗ Don't store plain text passwords
✗ Don't use MD5 or SHA1 for passwords
✗ Don't send passwords in URLs
✗ Don't log passwords
```

### API key security

```
✓ Generate cryptographically random keys
✓ Hash keys before storing (like passwords)
✓ Allow multiple keys per account
✓ Support key rotation
✓ Set expiration dates
✓ Log key usage
✓ Rate limit per key

✗ Don't expose keys in client-side code
✗ Don't put keys in URLs
✗ Don't share keys between environments
```

### Common vulnerabilities

**1. Broken authentication**
```
Problem: Weak passwords, no rate limiting
Fix: Strong password policy, rate limiting, MFA
```

**2. Broken authorization**
```
Problem: User can access other users' data
Fix: Always check resource ownership
```

**3. JWT vulnerabilities**
```
Problem: Algorithm confusion (none, HS256 vs RS256)
Fix: Explicitly specify and validate algorithm
```

**4. Session fixation**
```
Problem: Attacker sets session ID before login
Fix: Regenerate session ID after login
```

**5. CSRF**
```
Problem: Attacker tricks user into making requests
Fix: CSRF tokens, SameSite cookies
```

---

## Implementation examples

### Login endpoint

```http
POST /auth/login HTTP/1.1
Content-Type: application/json

{
  "email": "john@example.com",
  "password": "secret123"
}

HTTP/1.1 200 OK
Content-Type: application/json

{
  "access_token": "eyJhbGciOiJIUzI1NiIs...",
  "refresh_token": "dGhpcyBpcyBhIHJlZnJlc2g...",
  "token_type": "Bearer",
  "expires_in": 900
}
```

### Protected endpoint

```http
GET /users/me HTTP/1.1
Authorization: Bearer eyJhbGciOiJIUzI1NiIs...

HTTP/1.1 200 OK
Content-Type: application/json

{
  "id": "user_123",
  "email": "john@example.com",
  "name": "John Doe"
}
```

### Unauthorized response

```http
GET /users/me HTTP/1.1

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

### Forbidden response

```http
DELETE /users/456 HTTP/1.1
Authorization: Bearer eyJhbGciOiJIUzI1NiIs...

HTTP/1.1 403 Forbidden
Content-Type: application/json

{
  "error": {
    "code": "FORBIDDEN",
    "message": "You don't have permission to delete this user"
  }
}
```

---

## Key takeaways

1. **Authentication ≠ Authorization.** Know the difference.

2. **Use the right method for your use case:**
   - API keys for server-to-server
   - OAuth for third-party access
   - JWT for stateless APIs
   - Sessions for traditional web apps

3. **Keep tokens short-lived.** Use refresh tokens for long sessions.

4. **Always use HTTPS.** No exceptions.

5. **Validate everything.** Token signature, expiration, issuer, permissions.

6. **Implement proper authorization.** Don't just check if user is logged in - check if they can access the specific resource.

7. **Log authentication events.** You'll need them for security audits.

---

[Next: Rate Limiting & Throttling →](./09-rate-limiting-throttling.md)