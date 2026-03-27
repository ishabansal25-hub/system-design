# GraphQL Essentials

[← Back to API Design](./README.md) | [← Previous: API Documentation](./11-api-documentation.md)

---

GraphQL is a query language for APIs that lets clients request exactly the data they need. It was developed by Facebook in 2012 and open-sourced in 2015. It's not a replacement for REST - it's an alternative that solves different problems.

I've seen teams adopt GraphQL because it's trendy, then struggle with complexity they didn't need. I've also seen teams stick with REST when GraphQL would have saved them months of work. Know when to use each.

---

## What is GraphQL?

GraphQL is:
- A **query language** for your API
- A **runtime** for executing those queries
- A **type system** that describes your data

```graphql
# Client asks for exactly what it needs
query {
  user(id: "123") {
    name
    email
    orders(last: 5) {
      id
      total
      items {
        name
        quantity
      }
    }
  }
}
```

```json
// Server returns exactly that
{
  "data": {
    "user": {
      "name": "John Doe",
      "email": "john@example.com",
      "orders": [
        {
          "id": "order_1",
          "total": 99.99,
          "items": [
            {"name": "Widget", "quantity": 2}
          ]
        }
      ]
    }
  }
}
```

---

## GraphQL vs. REST

| Aspect | REST | GraphQL |
|--------|------|---------|
| Endpoints | Multiple (`/users`, `/orders`) | Single (`/graphql`) |
| Data fetching | Fixed response structure | Client specifies fields |
| Over-fetching | Common | Eliminated |
| Under-fetching | Common (N+1 requests) | Eliminated |
| Versioning | URL or header versioning | Schema evolution |
| Caching | HTTP caching built-in | Requires custom solution |
| Learning curve | Lower | Higher |

### The problems GraphQL solves

**Over-fetching:**
```
REST: GET /users/123
Returns: id, name, email, phone, address, preferences, created_at, updated_at...
Client only needed: name, email
```

**Under-fetching (N+1 problem):**
```
REST:
1. GET /users/123           → Get user
2. GET /users/123/orders    → Get orders
3. GET /orders/1/items      → Get items for order 1
4. GET /orders/2/items      → Get items for order 2
...

GraphQL:
1. Single query gets everything
```

---

## Core concepts

### Schema

The schema defines your API's types and operations.

```graphql
type User {
  id: ID!
  name: String!
  email: String!
  orders: [Order!]!
}

type Order {
  id: ID!
  total: Float!
  status: OrderStatus!
  items: [OrderItem!]!
  user: User!
}

type OrderItem {
  id: ID!
  product: Product!
  quantity: Int!
  price: Float!
}

enum OrderStatus {
  PENDING
  PROCESSING
  SHIPPED
  DELIVERED
  CANCELLED
}

type Query {
  user(id: ID!): User
  users(limit: Int, offset: Int): [User!]!
  order(id: ID!): Order
}

type Mutation {
  createUser(input: CreateUserInput!): User!
  updateUser(id: ID!, input: UpdateUserInput!): User!
  deleteUser(id: ID!): Boolean!
  createOrder(input: CreateOrderInput!): Order!
}

input CreateUserInput {
  name: String!
  email: String!
}

input UpdateUserInput {
  name: String
  email: String
}
```

### Queries

Read operations.

```graphql
# Simple query
query {
  user(id: "123") {
    name
    email
  }
}

# Query with variables
query GetUser($userId: ID!) {
  user(id: $userId) {
    name
    email
    orders {
      id
      total
    }
  }
}

# Variables
{
  "userId": "123"
}
```

### Mutations

Write operations.

```graphql
mutation CreateUser($input: CreateUserInput!) {
  createUser(input: $input) {
    id
    name
    email
  }
}

# Variables
{
  "input": {
    "name": "John Doe",
    "email": "john@example.com"
  }
}
```

### Subscriptions

Real-time updates.

```graphql
subscription OnOrderStatusChanged($orderId: ID!) {
  orderStatusChanged(orderId: $orderId) {
    id
    status
    updatedAt
  }
}
```

---

## Type system

### Scalar types

```graphql
type Example {
  id: ID!           # Unique identifier
  name: String!     # UTF-8 string
  age: Int!         # 32-bit integer
  price: Float!     # Double-precision float
  active: Boolean!  # true or false
}
```

### Custom scalars

```graphql
scalar DateTime
scalar JSON
scalar URL

type User {
  createdAt: DateTime!
  metadata: JSON
  website: URL
}
```

### Object types

```graphql
type User {
  id: ID!
  name: String!
  email: String!
}
```

### Lists and non-null

```graphql
type User {
  name: String      # Nullable string
  name: String!     # Non-null string
  tags: [String]    # Nullable list of nullable strings
  tags: [String!]   # Nullable list of non-null strings
  tags: [String]!   # Non-null list of nullable strings
  tags: [String!]!  # Non-null list of non-null strings
}
```

### Enums

```graphql
enum Status {
  ACTIVE
  INACTIVE
  PENDING
}

type User {
  status: Status!
}
```

### Interfaces

```graphql
interface Node {
  id: ID!
}

type User implements Node {
  id: ID!
  name: String!
}

type Order implements Node {
  id: ID!
  total: Float!
}
```

### Unions

```graphql
union SearchResult = User | Order | Product

type Query {
  search(query: String!): [SearchResult!]!
}
```

---

## Resolvers

Resolvers are functions that fetch data for each field.

```javascript
const resolvers = {
  Query: {
    user: async (parent, { id }, context) => {
      return context.db.users.findById(id);
    },
    users: async (parent, { limit, offset }, context) => {
      return context.db.users.findAll({ limit, offset });
    }
  },
  
  User: {
    orders: async (user, args, context) => {
      return context.db.orders.findByUserId(user.id);
    }
  },
  
  Mutation: {
    createUser: async (parent, { input }, context) => {
      return context.db.users.create(input);
    }
  }
};
```

---

## N+1 problem and DataLoader

### The problem

```graphql
query {
  users {        # 1 query for users
    name
    orders {     # N queries for orders (one per user)
      id
    }
  }
}
```

### The solution: DataLoader

```javascript
const DataLoader = require('dataloader');

// Batch function
const orderLoader = new DataLoader(async (userIds) => {
  const orders = await db.orders.findByUserIds(userIds);
  // Return orders in same order as userIds
  return userIds.map(id => orders.filter(o => o.userId === id));
});

// Resolver
const resolvers = {
  User: {
    orders: (user, args, context) => {
      return context.loaders.orders.load(user.id);
    }
  }
};
```

Now instead of N queries, we make 1 batched query.

---

## Pagination

### Offset-based

```graphql
type Query {
  users(limit: Int = 20, offset: Int = 0): [User!]!
}

query {
  users(limit: 20, offset: 40) {
    id
    name
  }
}
```

### Cursor-based (Relay-style)

```graphql
type Query {
  users(first: Int, after: String, last: Int, before: String): UserConnection!
}

type UserConnection {
  edges: [UserEdge!]!
  pageInfo: PageInfo!
  totalCount: Int!
}

type UserEdge {
  node: User!
  cursor: String!
}

type PageInfo {
  hasNextPage: Boolean!
  hasPreviousPage: Boolean!
  startCursor: String
  endCursor: String
}
```

```graphql
query {
  users(first: 20, after: "cursor123") {
    edges {
      node {
        id
        name
      }
      cursor
    }
    pageInfo {
      hasNextPage
      endCursor
    }
  }
}
```

---

## Error handling

### GraphQL errors

```json
{
  "data": {
    "user": null
  },
  "errors": [
    {
      "message": "User not found",
      "locations": [{"line": 2, "column": 3}],
      "path": ["user"],
      "extensions": {
        "code": "NOT_FOUND"
      }
    }
  ]
}
```

### Partial responses

GraphQL can return partial data with errors:

```json
{
  "data": {
    "user": {
      "name": "John",
      "orders": null
    }
  },
  "errors": [
    {
      "message": "Failed to fetch orders",
      "path": ["user", "orders"]
    }
  ]
}
```

### Error types

```javascript
const { ApolloError, UserInputError, AuthenticationError } = require('apollo-server');

const resolvers = {
  Query: {
    user: async (_, { id }, context) => {
      if (!context.user) {
        throw new AuthenticationError('Must be logged in');
      }
      
      const user = await db.users.findById(id);
      if (!user) {
        throw new ApolloError('User not found', 'NOT_FOUND');
      }
      
      return user;
    }
  },
  
  Mutation: {
    createUser: async (_, { input }) => {
      if (!input.email.includes('@')) {
        throw new UserInputError('Invalid email', {
          invalidArgs: ['email']
        });
      }
      return db.users.create(input);
    }
  }
};
```

---

## Authentication & Authorization

### Context

```javascript
const server = new ApolloServer({
  typeDefs,
  resolvers,
  context: ({ req }) => {
    const token = req.headers.authorization || '';
    const user = getUser(token);
    return { user, db, loaders };
  }
});
```

### Field-level authorization

```javascript
const resolvers = {
  User: {
    email: (user, args, context) => {
      // Only return email if user is viewing their own profile
      if (context.user?.id === user.id) {
        return user.email;
      }
      return null;
    }
  }
};
```

### Directive-based authorization

```graphql
directive @auth(requires: Role = USER) on FIELD_DEFINITION

enum Role {
  ADMIN
  USER
  GUEST
}

type Query {
  users: [User!]! @auth(requires: ADMIN)
  me: User @auth(requires: USER)
  publicPosts: [Post!]!
}
```

---

## Performance considerations

### Query complexity

Limit query depth and complexity:

```javascript
const depthLimit = require('graphql-depth-limit');
const { createComplexityLimitRule } = require('graphql-validation-complexity');

const server = new ApolloServer({
  typeDefs,
  resolvers,
  validationRules: [
    depthLimit(10),
    createComplexityLimitRule(1000)
  ]
});
```

### Persisted queries

Pre-register queries to reduce payload size:

```javascript
// Client sends hash instead of full query
{
  "extensions": {
    "persistedQuery": {
      "sha256Hash": "abc123..."
    }
  },
  "variables": {"id": "123"}
}
```

### Caching

```javascript
const resolvers = {
  Query: {
    user: async (_, { id }, context, info) => {
      info.cacheControl.setCacheHint({ maxAge: 60 });
      return db.users.findById(id);
    }
  }
};
```

---

## When to use GraphQL

### Good fit

- **Mobile apps** - Minimize data transfer, reduce round trips
- **Complex UIs** - Dashboard with data from many sources
- **Rapid iteration** - Frontend can evolve without backend changes
- **Multiple clients** - Different clients need different data shapes
- **Microservices** - Aggregate data from multiple services

### Not ideal

- **Simple CRUD** - REST is simpler
- **File uploads** - REST handles this better
- **Caching-heavy** - HTTP caching is easier with REST
- **Small team** - GraphQL has a learning curve
- **Public API** - REST is more widely understood

---

## GraphQL vs. REST decision tree

```
Do you have multiple clients with different data needs?
├── Yes → Consider GraphQL
└── No
    ├── Is your data highly interconnected?
    │   ├── Yes → Consider GraphQL
    │   └── No → REST is probably fine
    └── Do you need real-time updates?
        ├── Yes → GraphQL subscriptions or WebSockets
        └── No → REST is probably fine

Is your team experienced with GraphQL?
├── Yes → GraphQL is viable
└── No → Consider the learning curve
```

---

## Key takeaways

1. **GraphQL solves specific problems.** Over-fetching, under-fetching, and multiple round trips.

2. **It's not always better than REST.** Use the right tool for the job.

3. **The N+1 problem is real.** Use DataLoader or similar batching.

4. **Security matters.** Limit query depth and complexity.

5. **Caching is harder.** You lose HTTP caching benefits.

6. **Schema design is crucial.** Think carefully about your types and relationships.

7. **Start simple.** You don't need every GraphQL feature on day one.

---

[Next: gRPC & Protocol Buffers →](./13-grpc-protocol-buffers.md)