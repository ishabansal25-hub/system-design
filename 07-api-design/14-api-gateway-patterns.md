# API Gateway Patterns

[← Back to API Design](./README.md) | [← Previous: gRPC & Protocol Buffers](./13-grpc-protocol-buffers.md)

---

An API Gateway is the front door to your microservices. It sits between clients and your backend services, handling cross-cutting concerns like authentication, rate limiting, and routing. Without one, every service needs to implement these concerns independently.

I've seen organizations without an API gateway struggle with inconsistent authentication, duplicated rate limiting logic, and clients that need to know about every internal service. An API gateway solves these problems.

---

## What is an API Gateway?

```
Without API Gateway:
┌─────────┐     ┌─────────────┐
│ Client  │────▶│ User Service│
└─────────┘     └─────────────┘
     │          ┌─────────────┐
     └─────────▶│Order Service│
                └─────────────┘
     │          ┌─────────────┐
     └─────────▶│Product Svc  │
                └─────────────┘

With API Gateway:
┌─────────┐     ┌─────────────┐     ┌─────────────┐
│ Client  │────▶│ API Gateway │────▶│ User Service│
└─────────┘     └─────────────┘     └─────────────┘
                       │            ┌─────────────┐
                       └───────────▶│Order Service│
                       │            └─────────────┘
                       │            ┌─────────────┐
                       └───────────▶│Product Svc  │
                                    └─────────────┘
```

---

## Core responsibilities

### 1. Request routing

Route requests to appropriate backend services.

```yaml
routes:
  - path: /users/**
    service: user-service
    
  - path: /orders/**
    service: order-service
    
  - path: /products/**
    service: product-service
```

### 2. Authentication & Authorization

Verify identity and permissions before forwarding requests.

```
Client → Gateway → Verify JWT → Forward to service
                      ↓
                 Invalid? → 401 Unauthorized
```

### 3. Rate limiting

Protect backend services from overload.

```yaml
rate_limits:
  - path: /api/**
    limit: 1000
    window: 1m
    
  - path: /api/expensive/**
    limit: 10
    window: 1m
```

### 4. Load balancing

Distribute traffic across service instances.

```
Gateway → Load Balancer → Service Instance 1
                       → Service Instance 2
                       → Service Instance 3
```

### 5. Request/Response transformation

Modify requests and responses as needed.

```yaml
transformations:
  - path: /v1/users
    request:
      add_header: X-API-Version: 1
    response:
      remove_header: X-Internal-Id
```

### 6. Caching

Cache responses to reduce backend load.

```yaml
caching:
  - path: /products/**
    ttl: 5m
    methods: [GET]
```

### 7. Monitoring & Logging

Centralized observability.

```
All requests → Gateway → Metrics, Logs, Traces
```

---

## Gateway patterns

### Pattern 1: Simple proxy

Basic routing without transformation.

```
Client → Gateway → Service
```

**Use case:** Simple microservices, internal APIs

### Pattern 2: Backend for Frontend (BFF)

Separate gateways for different clients.

```
┌──────────────┐     ┌─────────────┐
│ Mobile App   │────▶│ Mobile BFF  │────┐
└──────────────┘     └─────────────┘    │
                                        ▼
┌──────────────┐     ┌─────────────┐  ┌─────────┐
│ Web App      │────▶│  Web BFF    │─▶│Services │
└──────────────┘     └─────────────┘  └─────────┘
                                        ▲
┌──────────────┐     ┌─────────────┐    │
│ Partner API  │────▶│Partner BFF  │────┘
└──────────────┘     └─────────────┘
```

**Benefits:**
- Optimized responses for each client
- Different authentication per client
- Independent evolution

### Pattern 3: Aggregation gateway

Combine multiple service calls into one.

```
Client: GET /dashboard

Gateway:
  1. GET /users/me
  2. GET /orders/recent
  3. GET /notifications/unread
  
Response: Combined data from all three
```

```javascript
// Gateway aggregation logic
async function getDashboard(userId) {
  const [user, orders, notifications] = await Promise.all([
    userService.getUser(userId),
    orderService.getRecentOrders(userId),
    notificationService.getUnread(userId)
  ]);
  
  return {
    user,
    recentOrders: orders,
    unreadNotifications: notifications
  };
}
```

### Pattern 4: Edge gateway

Gateway at the edge of your network (CDN integration).

```
Client → CDN/Edge → Gateway → Services
           ↓
        Cached responses
```

**Benefits:**
- Lower latency
- DDoS protection
- Global distribution

---

## Request routing strategies

### Path-based routing

```yaml
routes:
  - path: /api/v1/users/**
    service: user-service-v1
    
  - path: /api/v2/users/**
    service: user-service-v2
    
  - path: /api/orders/**
    service: order-service
```

### Header-based routing

```yaml
routes:
  - path: /api/users/**
    conditions:
      - header: X-API-Version
        value: "2"
    service: user-service-v2
    
  - path: /api/users/**
    service: user-service-v1  # Default
```

### Weight-based routing (canary)

```yaml
routes:
  - path: /api/users/**
    backends:
      - service: user-service-v1
        weight: 90
      - service: user-service-v2
        weight: 10  # 10% canary
```

### Content-based routing

```yaml
routes:
  - path: /api/process
    conditions:
      - body_json: $.type
        value: "premium"
    service: premium-processor
    
  - path: /api/process
    service: standard-processor
```

---

## Authentication patterns

### JWT validation

```
1. Client sends: Authorization: Bearer <JWT>
2. Gateway validates JWT signature
3. Gateway extracts claims
4. Gateway forwards request with user context
```

```yaml
authentication:
  jwt:
    issuer: https://auth.example.com
    audience: api.example.com
    jwks_uri: https://auth.example.com/.well-known/jwks.json
```

### API key validation

```
1. Client sends: X-API-Key: <key>
2. Gateway looks up key in database/cache
3. Gateway applies key's rate limits and permissions
4. Gateway forwards request
```

### OAuth token introspection

```
1. Client sends: Authorization: Bearer <token>
2. Gateway calls auth server to validate token
3. Auth server returns token info
4. Gateway forwards request with user context
```

### Pass-through authentication

```
1. Client sends credentials
2. Gateway forwards to auth service
3. Auth service validates and returns token
4. Gateway returns token to client
```

---

## Rate limiting at the gateway

### Global rate limiting

```yaml
rate_limits:
  global:
    limit: 10000
    window: 1s
```

### Per-client rate limiting

```yaml
rate_limits:
  per_client:
    limit: 100
    window: 1m
    key: client_id
```

### Per-endpoint rate limiting

```yaml
rate_limits:
  endpoints:
    - path: /api/search
      limit: 10
      window: 1m
      
    - path: /api/users
      limit: 100
      window: 1m
```

### Tiered rate limiting

```yaml
rate_limits:
  tiers:
    free:
      limit: 60
      window: 1m
    pro:
      limit: 600
      window: 1m
    enterprise:
      limit: 6000
      window: 1m
```

---

## Circuit breaker pattern

Prevent cascade failures when services are unhealthy.

```
States:
┌────────┐    failures > threshold    ┌────────┐
│ CLOSED │ ─────────────────────────▶ │  OPEN  │
└────────┘                            └────────┘
     ▲                                     │
     │                                     │ timeout
     │         ┌─────────────┐             │
     └─────────│ HALF-OPEN   │◀────────────┘
    success    └─────────────┘
```

```yaml
circuit_breaker:
  - service: user-service
    failure_threshold: 5
    timeout: 30s
    half_open_requests: 3
```

**Behavior:**
- **Closed:** Normal operation, requests pass through
- **Open:** Requests fail immediately (fast fail)
- **Half-open:** Limited requests to test recovery

---

## Response caching

### Cache configuration

```yaml
caching:
  - path: /api/products/**
    methods: [GET]
    ttl: 5m
    vary_by:
      - header: Accept-Language
      - query: currency
      
  - path: /api/users/me
    methods: [GET]
    ttl: 1m
    vary_by:
      - header: Authorization
```

### Cache invalidation

```yaml
cache_invalidation:
  - trigger:
      path: /api/products/**
      methods: [POST, PUT, DELETE]
    invalidate:
      - pattern: /api/products/**
```

---

## Request/Response transformation

### Add headers

```yaml
transformations:
  request:
    add_headers:
      X-Request-ID: ${uuid()}
      X-Forwarded-For: ${client_ip}
      
  response:
    add_headers:
      X-Response-Time: ${response_time}
    remove_headers:
      - X-Internal-Service
```

### Body transformation

```yaml
transformations:
  request:
    body:
      # Add field
      - set: $.metadata.gateway_timestamp
        value: ${timestamp()}
        
  response:
    body:
      # Remove internal fields
      - remove: $.internal_id
      - remove: $.debug_info
```

### Protocol translation

```
Client (REST) → Gateway → Service (gRPC)

Gateway translates:
- HTTP method → gRPC method
- JSON body → Protocol Buffer
- HTTP status → gRPC status
```

---

## Service discovery

### Static configuration

```yaml
services:
  user-service:
    url: http://user-service:8080
    
  order-service:
    url: http://order-service:8080
```

### DNS-based discovery

```yaml
services:
  user-service:
    dns: user-service.default.svc.cluster.local
    port: 8080
```

### Service registry (Consul, Eureka)

```yaml
service_discovery:
  type: consul
  address: consul:8500
  
services:
  user-service:
    service_name: user-service
```

---

## Popular API gateways

### Kong

Open-source, Lua-based, plugin architecture.

```yaml
# Kong configuration
services:
  - name: user-service
    url: http://user-service:8080
    routes:
      - paths: [/users]
    plugins:
      - name: rate-limiting
        config:
          minute: 100
      - name: jwt
```

### AWS API Gateway

Managed service, integrates with AWS ecosystem.

```yaml
# AWS SAM template
Resources:
  Api:
    Type: AWS::Serverless::Api
    Properties:
      StageName: prod
      Auth:
        DefaultAuthorizer: MyCognitoAuthorizer
```

### NGINX

High-performance, widely used.

```nginx
upstream user_service {
    server user-service-1:8080;
    server user-service-2:8080;
}

server {
    location /api/users {
        proxy_pass http://user_service;
        
        # Rate limiting
        limit_req zone=api burst=20;
        
        # Caching
        proxy_cache api_cache;
        proxy_cache_valid 200 5m;
    }
}
```

### Envoy

Modern, cloud-native, used in service meshes.

```yaml
static_resources:
  listeners:
    - address:
        socket_address:
          address: 0.0.0.0
          port_value: 8080
      filter_chains:
        - filters:
            - name: envoy.filters.network.http_connection_manager
              typed_config:
                route_config:
                  virtual_hosts:
                    - name: backend
                      routes:
                        - match:
                            prefix: "/users"
                          route:
                            cluster: user_service
```

---

## Best practices

### 1. Keep the gateway thin

```
Good: Authentication, rate limiting, routing
Bad: Business logic, complex transformations
```

### 2. Handle failures gracefully

```yaml
fallback:
  - path: /api/recommendations
    on_error:
      status: 200
      body: {"recommendations": []}  # Empty fallback
```

### 3. Use health checks

```yaml
health_checks:
  - service: user-service
    path: /health
    interval: 10s
    timeout: 5s
    unhealthy_threshold: 3
```

### 4. Implement proper timeouts

```yaml
timeouts:
  connect: 5s
  read: 30s
  write: 30s
  
  # Per-route overrides
  routes:
    - path: /api/reports/**
      read: 120s  # Long-running reports
```

### 5. Log and monitor everything

```yaml
logging:
  format: json
  fields:
    - request_id
    - client_ip
    - method
    - path
    - status
    - latency
    - upstream_service
```

---

## Key takeaways

1. **API Gateway is essential for microservices.** Centralizes cross-cutting concerns.

2. **Choose the right pattern.** Simple proxy, BFF, or aggregation based on needs.

3. **Implement circuit breakers.** Prevent cascade failures.

4. **Cache strategically.** Reduce backend load for read-heavy endpoints.

5. **Keep it thin.** Gateway handles infrastructure, not business logic.

6. **Monitor everything.** The gateway sees all traffic - use that visibility.

7. **Plan for failure.** Timeouts, retries, fallbacks.

---

[Next: Real-World API Examples →](./15-real-world-api-examples.md)