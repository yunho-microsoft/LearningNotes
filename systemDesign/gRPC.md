# gRPC — Interview Notes

> Based on: [gRPC official documentation](https://grpc.io/docs/) | [Protocol Buffers](https://protobuf.dev/) | [HTTP/2 spec (RFC 7540)](https://httpwg.org/specs/rfc7540.html)
>
> All links verified as of March 2026.

---

## What is gRPC?

**gRPC** (Google Remote Procedure Call) is an open-source, high-performance RPC (Remote Procedure Call) framework developed by Google. It allows a client to call functions on a remote server as if they were local function calls.

- Released publicly by Google in **2015**
- Now a **Cloud Native Computing Foundation (CNCF)** project
- Used heavily by Google, Netflix, Uber, Lyft, and others for internal microservice communication

Official docs: https://grpc.io/docs/what-is-grpc/introduction/

---

## How gRPC Differs from REST

| | REST (HTTP/1.1) | gRPC (HTTP/2) |
|---|---|---|
| Protocol | HTTP/1.1 (text) | HTTP/2 (binary) |
| Data format | JSON | Protocol Buffers (protobuf) |
| Contract | Optional (OpenAPI/Swagger) | Required (`.proto` file) |
| Streaming | Workarounds (SSE, WebSockets) | Native (4 modes) |
| Code generation | Manual or optional tooling | Auto-generated from `.proto` |
| Browser support | Universal | Requires `grpc-web` proxy |
| Debugging | Easy (`curl`, DevTools) | Needs tooling (`grpcurl`) |
| Best for | Public APIs, CRUD, web clients | Microservices, low-latency, streaming |

---

## HTTP/1.1 vs HTTP/2 — The Foundation

gRPC is built entirely on **HTTP/2**. Understanding the difference matters.

### HTTP/1.1 Limitations
- **Text-based** — verbose, slower to parse
- **One request at a time** per connection — causes head-of-line blocking
- Workaround: open 6–8 parallel TCP connections (wasteful)
- Headers repeated in full on every request

### HTTP/2 Improvements
- **Binary framing** — smaller, faster to parse
- **Multiplexing** — many concurrent streams over a single TCP connection, no blocking
- **Header compression** (HPACK) — headers sent once, only deltas after
- **Server push** — server can proactively send assets
- **Stream prioritization** — higher-priority requests processed first

> HTTP/2 spec: https://httpwg.org/specs/rfc7540.html

---

## Key Features of gRPC

### 1. Protocol Buffers (Protobuf)
- Binary serialization format — typically **5–10× smaller** than JSON
- Defined in `.proto` files — the single source of truth for the API contract
- Code is **auto-generated** for Go, Python, Java, Node.js, C++, and more from one `.proto` file
- Faster to serialize/deserialize than JSON

> Protobuf docs: https://protobuf.dev/

```protobuf
// Example .proto definition
syntax = "proto3";

service UserService {
  rpc GetUser (UserRequest) returns (UserResponse);
  rpc ListUsers (ListRequest) returns (stream UserResponse);
}

message UserRequest {
  string user_id = 1;
}

message UserResponse {
  string user_id = 1;
  string name = 2;
  string email = 3;
}
```

### 2. Four Communication Patterns

| Pattern | Description | Example use case |
|---|---|---|
| **Unary** | 1 request → 1 response (like REST) | Fetch a user record |
| **Server streaming** | 1 request → stream of responses | Live feed, file download |
| **Client streaming** | Stream of requests → 1 response | File upload, sensor data |
| **Bidirectional streaming** | Both sides stream simultaneously | Chat, multiplayer games |

### 3. Strongly Typed Contracts
- The `.proto` file is the contract — client and server must agree at **compile time**
- Eliminates integration bugs caused by mismatched API expectations
- Breaking changes are caught before deployment, not at runtime

### 4. Polyglot Code Generation
- Define once in `.proto`, generate client/server stubs in any supported language
- Especially powerful for systems where services are written in different languages (Go, Python, Java, etc.)

### 5. Built-in Features
- Authentication (TLS, token-based)
- Load balancing
- Health checking
- Deadline/timeout propagation across service calls
- Cancellation

---

## Limitations of gRPC

### 1. No Native Browser Support
- Browsers cannot make native gRPC calls — they don't expose the HTTP/2 framing controls gRPC requires
- **Workaround:** `grpc-web` — requires an Envoy proxy in front; loses some streaming features; adds operational complexity
- This is the **biggest blocker** for public-facing, browser-based APIs

### 2. Harder to Debug
- Binary protocol — you can't just read the payload in DevTools or with `curl`
- Requires tools like `grpcurl` or Postman's gRPC support
- REST's human-readable JSON is a real ergonomic advantage during development

### 3. `.proto` File Overhead
- Every service needs a `.proto` file
- Changes to the API require re-running code generation and re-shipping client stubs
- Client and server proto versions must stay in sync — version mismatch breaks calls
- Bad for **public APIs** consumed by third-party developers who'd need to run code generation before making any call

### 4. Infrastructure Compatibility
- Some older proxies, firewalls, and load balancers don't understand HTTP/2 framing
- REST/HTTP/1.1 is universally supported by CDNs, API gateways, and network infrastructure

### 5. Complexity Cost
- Adds a build step (protoc, code gen) to your CI/CD pipeline
- Overkill for simple CRUD apps where JSON serialization overhead is irrelevant compared to database query time

---

## The `.proto` File at Build Time vs Runtime

A common misconception: the `.proto` file is a **developer artifact**, not a runtime artifact. The browser never sees it.

**Build time (developer's machine / CI):**
1. Write `service.proto`
2. Run `protoc` (the protobuf compiler) to generate JS/Go/Python client stubs
3. Bundle the generated stubs into the app

**Runtime (in the browser/client):**
- Only the compiled stubs run — the `.proto` file itself is never shipped to users
- The browser calls `grpc-web` → Envoy proxy → real gRPC backend over HTTP/2

This is analogous to writing TypeScript but only shipping compiled JavaScript to users.

---

## When to Use gRPC

### Use gRPC when:

**1. Microservice-to-microservice communication**
The #1 real-world use case. When internal services call each other hundreds or thousands of times per second, binary encoding and persistent HTTP/2 connections compound into massive performance savings. You control both ends, so the `.proto` contract is a feature — it enforces API agreement at compile time across teams.

**2. Real-time / bidirectional streaming**
Chat apps, live dashboards, multiplayer games, stock tickers — anything needing a persistent firehose of data in both directions. gRPC handles this natively; REST requires cobbling together WebSockets + SSE as workarounds.

**3. Polyglot systems**
Multiple services written in different languages (Go, Python, Java, Node.js). Define the contract once in `.proto`, generate idiomatic clients for every language automatically.

**4. Mobile backends**
Protobuf payloads are 5–10× smaller than JSON — meaningful on slow mobile networks and in bandwidth-constrained regions.

**5. IoT / embedded devices**
Sensors and edge devices have limited CPU and memory. Parsing JSON is expensive at that scale. Protobuf's binary format is far more efficient.

**6. ML inference serving**
Real-time predictions (fraud detection, recommendations, autocomplete) need low-latency calls. TensorFlow Serving and NVIDIA Triton Inference Server both use gRPC as their primary protocol.

### Use REST when:

- Building a **public API** for third-party developers
- Your clients are **browsers** (without grpc-web complexity)
- You need **simple CRUD** — the performance overhead of JSON is irrelevant vs your DB query time
- You want **maximum debuggability** and ease of integration
- Your infra (CDNs, gateways) isn't HTTP/2 compatible

---

## The Signal Question

> **Do you control both ends of the connection, and does performance actually matter at your scale?**

- **Yes to both** → gRPC is worth the complexity cost
- **No / unsure** → REST is the safer, simpler default

---

## Real-World Architecture Pattern

The dominant pattern at companies like Google, Netflix, and Uber:

```
Browser / Mobile
      |
      | REST + JSON (public surface)
      |
   API Gateway
      |
      | gRPC (internal network)
      |
   ┌──┴──────────────────┐
   │                      │
Auth Service    Orders Service    Notifications Service
   │                      │
   └──────── gRPC ────────┘
```

- **REST on the outside** — browser and mobile compatible, easy for third parties
- **gRPC on the inside** — high performance, strict contracts, streaming between microservices

---

## Quick Reference: Key Terms

| Term | Definition |
|---|---|
| **RPC** | Remote Procedure Call — calling a function on another machine as if it's local |
| **Protobuf** | Protocol Buffers — Google's binary serialization format used by gRPC |
| `.proto` file | Schema file that defines service methods and message types |
| **protoc** | The protobuf compiler that generates client/server code from `.proto` files |
| **Multiplexing** | Multiple concurrent request/response streams over a single TCP connection (HTTP/2) |
| **grpc-web** | A proxy layer that allows browsers to call gRPC services (limited support) |
| **Unary RPC** | Standard request/response — the gRPC equivalent of a REST call |
| **Streaming RPC** | One or both sides send a sequence of messages over an open connection |
| **HPACK** | HTTP/2 header compression algorithm |

---

## Official Resources

| Resource | Link |
|---|---|
| gRPC official docs | https://grpc.io/docs/ |
| Introduction to gRPC | https://grpc.io/docs/what-is-grpc/introduction/ |
| Core concepts & architecture | https://grpc.io/docs/what-is-grpc/core-concepts/ |
| gRPC guides | https://grpc.io/docs/guides/ |
| Who is using gRPC | https://grpc.io/about/ |
| Protocol Buffers | https://protobuf.dev/ |
| Protobuf overview | https://protobuf.dev/overview/ |
| Protobuf language guide (proto3) | https://protobuf.dev/programming-guides/proto3/ |
| HTTP/2 spec (RFC 7540) | https://httpwg.org/specs/rfc7540.html |
| gRPC GitHub | https://github.com/grpc/grpc |
| grpc-web GitHub | https://github.com/grpc/grpc-web |