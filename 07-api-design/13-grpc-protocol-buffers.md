# gRPC & Protocol Buffers

[← Back to API Design](./README.md) | [← Previous: GraphQL Essentials](./12-graphql-essentials.md)

---

gRPC is a high-performance RPC (Remote Procedure Call) framework developed by Google. It uses Protocol Buffers for serialization and HTTP/2 for transport. When you need speed and efficiency, especially for internal service-to-service communication, gRPC is often the answer.

I've seen gRPC reduce latency by 10x compared to REST+JSON in high-throughput microservices. But I've also seen teams struggle with it because they didn't understand the tradeoffs.

---

## What is gRPC?

gRPC is:
- **RPC framework** - Call remote functions like local functions
- **Protocol Buffers** - Binary serialization format
- **HTTP/2** - Multiplexed, bidirectional streaming
- **Language agnostic** - Generate clients in any language

```protobuf
// Define your service
service UserService {
  rpc GetUser(GetUserRequest) returns (User);
  rpc CreateUser(CreateUserRequest) returns (User);
  rpc ListUsers(ListUsersRequest) returns (stream User);
}
```

---

## gRPC vs. REST

| Aspect | REST | gRPC |
|--------|------|------|
| Protocol | HTTP/1.1 or HTTP/2 | HTTP/2 |
| Payload | JSON (text) | Protocol Buffers (binary) |
| Contract | OpenAPI (optional) | .proto files (required) |
| Streaming | Limited | Built-in |
| Browser support | Native | Requires proxy |
| Human readable | Yes | No (binary) |
| Performance | Good | Excellent |
| Learning curve | Lower | Higher |

### When to use gRPC

**Good fit:**
- Microservices communication
- Low-latency, high-throughput systems
- Polyglot environments
- Real-time streaming
- Mobile clients (bandwidth-sensitive)

**Not ideal:**
- Public APIs (REST is more accessible)
- Browser clients (needs gRPC-Web)
- Simple CRUD applications
- When human readability matters

---

## Protocol Buffers

### Basic syntax

```protobuf
syntax = "proto3";

package example.user.v1;

option java_package = "com.example.user.v1";
option go_package = "example.com/user/v1";

message User {
  string id = 1;
  string name = 2;
  string email = 3;
  int32 age = 4;
  repeated string tags = 5;
  Address address = 6;
  UserStatus status = 7;
  google.protobuf.Timestamp created_at = 8;
}

message Address {
  string street = 1;
  string city = 2;
  string country = 3;
  string postal_code = 4;
}

enum UserStatus {
  USER_STATUS_UNSPECIFIED = 0;
  USER_STATUS_ACTIVE = 1;
  USER_STATUS_INACTIVE = 2;
  USER_STATUS_SUSPENDED = 3;
}
```

### Field types

| Proto Type | Default | Notes |
|------------|---------|-------|
| `double` | 0 | 64-bit float |
| `float` | 0 | 32-bit float |
| `int32` | 0 | Variable-length encoding |
| `int64` | 0 | Variable-length encoding |
| `uint32` | 0 | Unsigned |
| `uint64` | 0 | Unsigned |
| `sint32` | 0 | Signed (efficient for negative) |
| `sint64` | 0 | Signed (efficient for negative) |
| `fixed32` | 0 | Always 4 bytes |
| `fixed64` | 0 | Always 8 bytes |
| `bool` | false | |
| `string` | "" | UTF-8 |
| `bytes` | empty | Arbitrary bytes |

### Field numbers

Field numbers are critical - they identify fields in the binary format.

```protobuf
message User {
  string id = 1;      // Field number 1
  string name = 2;    // Field number 2
  string email = 3;   // Field number 3
  // Field 4 was removed - DON'T REUSE
  reserved 4;
  reserved "old_field";
  string phone = 5;   // Field number 5
}
```

**Rules:**
- Numbers 1-15 use 1 byte (use for frequent fields)
- Numbers 16-2047 use 2 bytes
- Never reuse field numbers
- Use `reserved` for removed fields

### Nested messages

```protobuf
message Order {
  string id = 1;
  repeated OrderItem items = 2;
  
  message OrderItem {
    string product_id = 1;
    int32 quantity = 2;
    double price = 3;
  }
}
```

### Oneof (union types)

```protobuf
message Payment {
  string id = 1;
  double amount = 2;
  
  oneof payment_method {
    CreditCard credit_card = 3;
    BankTransfer bank_transfer = 4;
    Wallet wallet = 5;
  }
}

message CreditCard {
  string number = 1;
  string expiry = 2;
}

message BankTransfer {
  string account = 1;
  string routing = 2;
}

message Wallet {
  string wallet_id = 1;
}
```

### Maps

```protobuf
message User {
  string id = 1;
  map<string, string> metadata = 2;
  map<string, Address> addresses = 3;
}
```

### Well-known types

```protobuf
import "google/protobuf/timestamp.proto";
import "google/protobuf/duration.proto";
import "google/protobuf/wrappers.proto";
import "google/protobuf/any.proto";

message Event {
  google.protobuf.Timestamp created_at = 1;
  google.protobuf.Duration duration = 2;
  google.protobuf.Int32Value nullable_count = 3;  // Nullable int
  google.protobuf.Any payload = 4;  // Any message type
}
```

---

## Service definitions

```protobuf
service UserService {
  // Unary RPC
  rpc GetUser(GetUserRequest) returns (User);
  rpc CreateUser(CreateUserRequest) returns (User);
  rpc UpdateUser(UpdateUserRequest) returns (User);
  rpc DeleteUser(DeleteUserRequest) returns (google.protobuf.Empty);
  
  // Server streaming
  rpc ListUsers(ListUsersRequest) returns (stream User);
  
  // Client streaming
  rpc UploadUsers(stream User) returns (UploadUsersResponse);
  
  // Bidirectional streaming
  rpc Chat(stream ChatMessage) returns (stream ChatMessage);
}

message GetUserRequest {
  string id = 1;
}

message CreateUserRequest {
  string name = 1;
  string email = 2;
}

message ListUsersRequest {
  int32 page_size = 1;
  string page_token = 2;
}
```

---

## RPC types

### Unary RPC

Simple request-response.

```protobuf
rpc GetUser(GetUserRequest) returns (User);
```

```python
# Client
response = stub.GetUser(GetUserRequest(id="123"))
print(response.name)
```

### Server streaming

Server sends multiple responses.

```protobuf
rpc ListUsers(ListUsersRequest) returns (stream User);
```

```python
# Client
for user in stub.ListUsers(ListUsersRequest(page_size=100)):
    print(user.name)
```

### Client streaming

Client sends multiple requests.

```protobuf
rpc UploadUsers(stream User) returns (UploadUsersResponse);
```

```python
# Client
def generate_users():
    for i in range(100):
        yield User(name=f"User {i}")

response = stub.UploadUsers(generate_users())
print(f"Uploaded {response.count} users")
```

### Bidirectional streaming

Both sides stream.

```protobuf
rpc Chat(stream ChatMessage) returns (stream ChatMessage);
```

```python
# Client
def chat():
    yield ChatMessage(text="Hello")
    yield ChatMessage(text="How are you?")

for response in stub.Chat(chat()):
    print(response.text)
```

---

## Error handling

### Status codes

| Code | Name | Description |
|------|------|-------------|
| 0 | OK | Success |
| 1 | CANCELLED | Operation cancelled |
| 2 | UNKNOWN | Unknown error |
| 3 | INVALID_ARGUMENT | Invalid argument |
| 4 | DEADLINE_EXCEEDED | Timeout |
| 5 | NOT_FOUND | Resource not found |
| 6 | ALREADY_EXISTS | Resource already exists |
| 7 | PERMISSION_DENIED | Permission denied |
| 8 | RESOURCE_EXHAUSTED | Rate limited |
| 9 | FAILED_PRECONDITION | Precondition failed |
| 10 | ABORTED | Operation aborted |
| 11 | OUT_OF_RANGE | Out of range |
| 12 | UNIMPLEMENTED | Not implemented |
| 13 | INTERNAL | Internal error |
| 14 | UNAVAILABLE | Service unavailable |
| 15 | DATA_LOSS | Data loss |
| 16 | UNAUTHENTICATED | Not authenticated |

### Returning errors

```python
from grpc import StatusCode

def GetUser(self, request, context):
    user = db.get_user(request.id)
    if not user:
        context.set_code(StatusCode.NOT_FOUND)
        context.set_details(f"User {request.id} not found")
        return User()
    return user
```

### Rich error details

```protobuf
import "google/rpc/status.proto";
import "google/rpc/error_details.proto";

// Error with details
google.rpc.Status {
  code: 3  // INVALID_ARGUMENT
  message: "Validation failed"
  details: [
    google.rpc.BadRequest {
      field_violations: [
        {field: "email", description: "Invalid email format"}
      ]
    }
  ]
}
```

---

## Authentication

### Token-based

```python
# Client - add metadata
metadata = [('authorization', f'Bearer {token}')]
response = stub.GetUser(request, metadata=metadata)

# Server - check metadata
def GetUser(self, request, context):
    metadata = dict(context.invocation_metadata())
    token = metadata.get('authorization', '').replace('Bearer ', '')
    if not validate_token(token):
        context.abort(StatusCode.UNAUTHENTICATED, 'Invalid token')
    # ...
```

### Interceptors

```python
class AuthInterceptor(grpc.ServerInterceptor):
    def intercept_service(self, continuation, handler_call_details):
        metadata = dict(handler_call_details.invocation_metadata)
        token = metadata.get('authorization', '')
        
        if not validate_token(token):
            return self._abort_handler
        
        return continuation(handler_call_details)
```

---

## Best practices

### API design

```protobuf
// Use request/response wrappers
rpc GetUser(GetUserRequest) returns (GetUserResponse);

message GetUserRequest {
  string id = 1;
}

message GetUserResponse {
  User user = 1;
}

// Not this
rpc GetUser(string) returns (User);  // Bad: no room for expansion
```

### Versioning

```protobuf
// Package versioning
package example.user.v1;

service UserService {
  rpc GetUser(GetUserRequest) returns (User);
}

// Later...
package example.user.v2;

service UserService {
  rpc GetUser(GetUserRequest) returns (User);  // Different implementation
}
```

### Backward compatibility

**Safe changes:**
- Add new fields (with new field numbers)
- Add new services or methods
- Add new enum values
- Change field names (numbers stay same)

**Breaking changes:**
- Remove fields
- Change field numbers
- Change field types
- Remove services or methods

---

## Performance tips

### Connection pooling

```python
# Reuse channels
channel = grpc.insecure_channel('localhost:50051')
stub = UserServiceStub(channel)

# Use for multiple requests
for i in range(1000):
    stub.GetUser(GetUserRequest(id=str(i)))
```

### Compression

```python
# Enable compression
channel = grpc.insecure_channel(
    'localhost:50051',
    compression=grpc.Compression.Gzip
)
```

### Deadlines

```python
# Set deadline
try:
    response = stub.GetUser(
        request,
        timeout=5.0  # 5 seconds
    )
except grpc.RpcError as e:
    if e.code() == StatusCode.DEADLINE_EXCEEDED:
        print("Request timed out")
```

---

## Key takeaways

1. **gRPC excels at service-to-service communication.** High performance, strong typing.

2. **Protocol Buffers are the contract.** Define your API in .proto files.

3. **Field numbers are forever.** Never reuse them.

4. **Use streaming for large data.** Server, client, or bidirectional.

5. **Handle errors properly.** Use appropriate status codes.

6. **Not for browsers.** Use gRPC-Web or REST for web clients.

7. **Backward compatibility matters.** Plan for schema evolution.

---

[Next: API Gateway Patterns →](./14-api-gateway-patterns.md)