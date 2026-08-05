# gRPC

> **In one line —** calling a function on another machine as if it were local: binary, typed, contract-first, and much faster than JSON over HTTP.

| | |
|---|---|
| **Full name** | gRPC Remote Procedure Call |
| **Category** | Communication Protocol |
| **Architectural Layer** | Application (over HTTP/2) |
| **Serialisation** | Protocol Buffers (protobuf) |
| **Related notes** | [HTTP HTTPS](HTTP%20HTTPS.md) · [API Design](API%20Design.md) · [Microservices](../10%20-%20Distributed%20Systems/Microservices.md) · [WebSockets](WebSockets.md) |

---

## 1. Short Definition

*What is it?*

gRPC is a framework for calling methods on a remote service as though they were local functions. You define the service in a `.proto` file, and gRPC generates typed client and server code in your language. Messages travel as compact binary over [HTTP/2](HTTP%20HTTPS.md).

---

## 2. Purpose

*What is its main purpose?*

To make service-to-service communication **fast, typed and contract-driven**, eliminating the guesswork and hand-written glue that REST-plus-JSON leaves to each developer.

---

## 3. Problem

*What engineering problem does it solve?*

REST with JSON has real costs at scale:

- **No enforced contract** — nothing prevents a field from silently changing shape
- **Verbose** — field names repeat in every message
- **Slow to parse** — text serialisation is expensive at high volume
- **Manual client code** — every consumer writes its own HTTP calls and parsing

```text
JSON      {"userId": 12345, "isActive": true}     ~40 bytes, parsed as text
Protobuf  [binary]                                ~8 bytes, parsed as a struct
```

---

## 4. Architecture Position

```text
Service A                          Service B
    ↓                                  ↑
Generated stub  ←──── .proto ────►  Generated stub
    ↓                                  ↑
gRPC runtime                       gRPC runtime
    ↓                                  ↑
HTTP/2  (multiplexed, one connection)
    ↓
TLS → TCP → IP
```

> [!IMPORTANT]
> gRPC is designed for **internal service-to-service** traffic. It sits between your microservices, not usually between your API and a browser.

---

## 5. The contract

```protobuf
service UserService {
  rpc GetUser (GetUserRequest) returns (User);
  rpc ListUsers (ListUsersRequest) returns (stream User);
}

message GetUserRequest {
  int64 user_id = 1;
}

message User {
  int64 id = 1;
  string name = 2;
  bool is_active = 3;
}
```

From this one file, gRPC generates client and server code for Go, Python, Java, C#, TypeScript and more. **The contract is the source of truth**, not documentation that drifts from reality.

---

## 6. The four call types

```text
UNARY               request → response                (like a normal function call)
SERVER STREAMING    request → many responses          (live feed, large result set)
CLIENT STREAMING    many requests → response          (upload, batch ingest)
BIDIRECTIONAL       both stream freely                (chat, real-time sync)
```

This is one of gRPC's genuine advantages: streaming is built in, not bolted on.

---

## 7. Real World Example

- **Google** built gRPC from its internal RPC system; essentially all internal Google service traffic uses this model.
- **Kubernetes** components communicate over gRPC, and its API uses protobuf internally.
- **Netflix, Square, Cloudflare** use it for internal microservice communication where JSON overhead became measurable.
- **Envoy and service meshes** speak gRPC for their control planes.

---

## 8. Input, Processing, Output

**Input:** a typed request object, constructed from generated code — a compile error if you get the shape wrong.

**Processing:** serialised to protobuf binary, sent over a multiplexed HTTP/2 stream, deserialised into a typed object on the other side.

**Output:** a typed response, or a gRPC status code (`NOT_FOUND`, `PERMISSION_DENIED`, `DEADLINE_EXCEEDED`).

---

## 9. gRPC vs REST

| | gRPC | REST + JSON |
|---|---|---|
| **Format** | Binary (protobuf) | Text (JSON) |
| **Contract** | Enforced by `.proto` | Convention, or OpenAPI |
| **Speed** | ~5–10× faster to serialise | Slower |
| **Size** | ~3–10× smaller | Larger |
| **Streaming** | Built in, four modes | Awkward (SSE, WebSockets) |
| **Browser support** | Needs gRPC-Web + a proxy | Native |
| **Human-readable** | No — needs tooling | Yes — `curl` works |
| **Caching** | No HTTP caching | Full HTTP caching |

---

## 10. When To Use

> [!TIP]
> Use gRPC for **internal service-to-service communication** in a polyglot system, especially where call volume is high, latency matters, or streaming is needed.

---

## 11. When NOT To Use

> [!CAUTION]
> - **Public APIs** — third-party developers expect REST and JSON. gRPC raises the barrier to integration considerably.
> - **Browser clients** — requires gRPC-Web and a translating proxy; usually not worth it.
> - **When you need HTTP caching or CDNs** — binary POST-like calls cannot be cached by intermediaries.
> - **Small systems** — two services do not justify a code-generation pipeline.
> - **When debuggability matters most** — you cannot `curl` a gRPC endpoint and read the answer.

---

## 12. Advantages and Disadvantages

**Advantages**
- Significantly faster and smaller on the wire
- A real, enforced, versionable contract
- Generated clients in many languages — no hand-written HTTP glue
- First-class streaming in all four directions
- HTTP/2 multiplexing: many concurrent calls on one connection
- Deadlines and cancellation built into the protocol

**Disadvantages**
- Not human-readable; debugging needs `grpcurl` or similar
- Poor browser support without a proxy
- Build complexity — code generation must be part of CI
- Weaker ecosystem of general-purpose tooling than HTTP
- Load balancing needs care: HTTP/2 keeps one connection open, so naive L4 balancers send everything to one backend

---

## 13. Performance Impact

| Resource | Impact |
|---|---|
| **Speed** | 5–10× faster serialisation than JSON |
| **Memory** | Lower — no intermediate text representation |
| **CPU** | Substantially less parsing work at high call volume |
| **Network** | 3–10× smaller payloads, plus header compression |

> [!TIP]
> The difference is negligible at 10 requests per second and decisive at 100,000. Choose gRPC because of your call volume, not because of a benchmark you read.

---

## 14. Security Considerations

> [!CAUTION]
> gRPC does not authenticate anything by default. Because it is typically used **inside** a network, it is easy to assume the network is trusted — which is exactly the assumption zero-trust architecture exists to remove.

- **Use TLS**, and prefer **mTLS** so both sides prove their identity
- **Authorise every call** — a valid connection is not a permission
- **Set deadlines on every call**; without them a hung dependency cascades through the system
- **Limit message size** — protobuf will happily deserialise something enormous
- **Reflection** exposes your entire service definition; disable it in production
- Validate inputs exactly as you would in REST — the generated types check *shape*, not *validity*

---

## 15. Mental Model

> [!NOTE]
> **REST is writing a letter in plain language; gRPC is a direct phone line with an agreed script.**
>
> The letter can be read by anyone and needs no preparation, but both sides must interpret it. The phone line is faster and neither side can misunderstand the format — but you cannot join the call without the script, and an outsider hearing it would understand nothing.

---

## 16. Mini Architecture Diagram

```text
                    .proto contract
                  ↙               ↘
        Generated client      Generated server
              ↓                     ↑
        Order Service  ──gRPC──►  User Service
              ↓                     ↑
                    HTTP/2 + TLS
                          ↓
              Browser ──REST/JSON──► API Gateway ──gRPC──► internal services
```

> [!TIP]
> This hybrid is the common production pattern: **REST at the edge for the outside world, gRPC internally** where speed and contracts matter.

---

## 17. Complete Request Flow

```text
Order Service calls userClient.GetUser(id: 42)
    ↓
Generated stub serialises to protobuf binary  (~8 bytes)
    ↓
Sent as a stream on an EXISTING HTTP/2 connection — no new handshake
    ↓
Deadline attached: 200 ms
    ↓
User Service deserialises into a typed struct
    ↓
Handler runs, queries the database
    ↓
Response serialised and streamed back
    ↓
Order Service receives a typed User object
    ↓
Deadline exceeded instead?  → DEADLINE_EXCEEDED status, call cancelled on both sides
```

---

## 18. Key Takeaway

> [!IMPORTANT]
> gRPC trades human readability and browser reach for speed, enforced contracts and built-in streaming — an excellent trade internally and usually a poor one at a public edge.

---

## 19. Common Mistakes

- **Exposing gRPC as a public API** and confusing every integrator
- **Calling without deadlines**, so a slow dependency hangs the whole chain
- **Naive L4 load balancing** — HTTP/2 reuses one connection, so traffic pools on one backend
- **Breaking the contract** by reusing or renumbering protobuf field numbers
- **Leaving reflection enabled** in production
- **Assuming the internal network is safe** and skipping mTLS
- **Not committing generated code or generating it in CI**, causing version drift between services

---

## 20. Open Source Technologies

- **gRPC** — implementations for Go, Java, Python, C#, Node, Rust
- **Protocol Buffers** — the serialisation format
- **buf** — modern protobuf tooling, linting and breaking-change detection
- **grpcurl**, **grpcui** — the `curl` and browser equivalents for gRPC
- **Envoy**, **Linkerd** — gRPC-aware load balancing and mTLS
- **ConnectRPC** — gRPC-compatible with better browser support

---

## 21. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 22. Workbook Exercise

- [ ] Write a `.proto` file for one existing endpoint of yours and generate a client from it.
- [ ] Compare the payload size of one JSON response with its protobuf equivalent.
- [ ] Explain in two sentences why gRPC load balancing needs an L7 proxy rather than a plain TCP balancer.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
.proto contract
    ↓
Generated stubs
    ↓
gRPC over HTTP/2 + TLS
    ↓
Service ←→ Service
```

## 2. Request Flow

```text
Input       a typed request object built from generated code
    ↓
Processing  protobuf serialisation → HTTP/2 stream → deserialisation → handler
    ↓
Output      a typed response, or a gRPC status code
```

## 3. Real-World Usage

**Kubernetes** uses gRPC and protobuf for internal component communication. At the scale a control plane operates — constant status updates from thousands of nodes — JSON parsing overhead would be a genuine bottleneck.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | A contract-first, binary RPC framework over HTTP/2 |
| **Why does it exist?** | Because REST plus JSON is verbose, untyped and slow at high volume |
| **Where does it belong?** | Between internal services, behind a REST edge |
| **When should I use it?** | High-volume internal communication — not for public or browser-facing APIs |
