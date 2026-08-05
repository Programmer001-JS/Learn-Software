# WebSockets

> **In one line —** a persistent two-way connection between browser and server, so the server can finally speak first.

| | |
|---|---|
| **Category** | Application Protocol |
| **Architectural Layer** | Application (over TCP) |
| **Scheme** | `ws://` · `wss://` (encrypted) |
| **Related notes** | [HTTP HTTPS](HTTP%20HTTPS.md) · [TCP IP](TCP%20IP.md) · [Event Loop](../05%20-%20Frontend%20Architecture/Event%20Loop.md) · [Pub Sub](../08%20-%20Databases%20and%20Data/Pub%20Sub.md) · [Redis](../08%20-%20Databases%20and%20Data/Redis.md) |

---

## 1. Short Definition

*What is it?*

A WebSocket is a long-lived, full-duplex connection between a client and a server. After an initial [HTTP](HTTP%20HTTPS.md) handshake, the connection is upgraded and both sides can send messages at any time, independently.

---

## 2. Purpose

*What is its main purpose?*

To let the **server push** data to the client without being asked. HTTP cannot do this: the client must always ask first.

---

## 3. Problem

*What engineering problem does it solve?*

Before WebSockets, real-time updates meant polling:

```text
POLLING                              WEBSOCKET

Client: "anything new?"  → no        Connection opened once
Client: "anything new?"  → no            ↓
Client: "anything new?"  → no        Server: "here is an update"  ← whenever it happens
Client: "anything new?"  → YES           ↓
                                     Server: "here is another"
99% of requests wasted               zero wasted requests
new TCP+TLS handshake each time      one connection, kept open
seconds of latency                   milliseconds
```

---

## 4. Architecture Position

```text
Browser / client
    ↓
WebSocket           ← you are here: persistent, bidirectional
    ↓
TLS  (wss://)
    ↓
TCP
    ↓
IP
```

Behind the server, a WebSocket layer almost always needs a **[pub/sub](../08%20-%20Databases%20and%20Data/Pub%20Sub.md) backend** — see section 12.

---

## 5. The upgrade handshake

WebSockets start as an ordinary HTTP request, which is what lets them work through existing proxies and firewalls.

```text
Client → GET /socket HTTP/1.1
         Upgrade: websocket
         Connection: Upgrade
         Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==

Server ← HTTP/1.1 101 Switching Protocols
         Upgrade: websocket
         Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=

         ══ from here on: raw WebSocket frames, both directions ══
```

---

## 6. Real World Example

- **Chat applications** — Slack, WhatsApp Web, Discord.
- **Live dashboards and trading** — price updates pushed as they happen.
- **Collaborative editing** — Google Docs, Figma: every keystroke is an event.
- **Multiplayer games** — low-latency state synchronisation.
- **Notifications** — "someone commented on your post", delivered instantly.

---

## 7. Input, Processing, Output

**Input:** messages — text (usually JSON) or binary — from either side, at any time.

**Processing:** each message is wrapped in a small frame (2–14 bytes of overhead, compared with hundreds of bytes of HTTP headers) and sent over the existing connection.

**Output:** an event delivered to the other side's handler, typically within milliseconds.

---

## 8. Internal Idea

*How does it work internally?*

Underneath, it is simply a TCP connection that never closes. The critical consequence is **state**: the server must remember every open connection.

> [!IMPORTANT]
> This breaks HTTP's statelessness, and that is the single most important architectural implication. A user connected to server A cannot be reached by server B — which is why scaling WebSockets requires a shared message backbone.

---

## 9. Communication and Dependencies

- **Client** — the browser's native `WebSocket` API, or a library
- **Server** — must support long-lived connections (Node.js, Go, ASP.NET Core, FastAPI, Phoenix)
- **Load balancer** — must support WebSocket upgrade and sticky routing
- **[Redis](../08%20-%20Databases%20and%20Data/Redis.md) pub/sub** or similar — to broadcast across multiple server instances

---

## 10. Alternatives

```text
Polling               simple, wasteful, high latency
    ↓
Long polling          hold the request open until there is news — better, still clumsy
    ↓
Server-Sent Events    server→client only, plain HTTP, auto-reconnect built in
    ↓
WebSocket             full duplex, low overhead
    ↓
WebRTC                peer-to-peer, for audio, video and direct data
```

> [!TIP]
> If updates only flow **server → client** (notifications, live feeds, progress bars), **Server-Sent Events are simpler**: plain HTTP, automatic reconnection, no special proxy configuration. Reach for WebSockets when the client genuinely needs to push too.

---

## 11. When To Use

> [!TIP]
> Use WebSockets when updates are frequent, bidirectional and latency-sensitive: chat, collaboration, live trading, multiplayer, real-time dashboards.

---

## 12. When NOT To Use

> [!CAUTION]
> - **Infrequent updates** — a connection held open for hours to deliver two messages is wasteful. Poll, or use SSE.
> - **Request/response semantics** — WebSockets have no built-in correlation between a request and its reply; you end up rebuilding HTTP badly.
> - **When you need HTTP caching** — WebSocket traffic is invisible to caches and CDNs.
> - **Serverless platforms** — most bill and scale per short request; long-lived connections fit poorly and are expensive.

---

## 13. Scaling — the part that surprises people

```text
ONE SERVER                        MULTIPLE SERVERS

User A ─┐                         User A ── Server 1
        ├── Server                User B ── Server 2
User B ─┘                              ↓
                                  A messages B — Server 1 has no connection to B
in-memory broadcast works             ↓
                                  Server 1 → REDIS PUB/SUB → Server 2 → User B
```

> [!IMPORTANT]
> The moment you run a second instance, you need a shared pub/sub backbone. This is not optional and it is the most common thing teams discover too late.

---

## 14. Advantages and Disadvantages

**Advantages**
- True bidirectional, low-latency communication
- Tiny per-message overhead compared with HTTP
- One connection instead of thousands of requests
- Supported natively by every browser

**Disadvantages**
- **Stateful** — breaks the easy horizontal scaling HTTP gives you
- Connections consume server memory and file descriptors
- Reconnection, backoff and message replay must be implemented by you
- No caching, no standard status codes, weaker tooling
- Some corporate proxies still interfere with long-lived connections

---

## 15. Performance Impact

| Resource | Impact |
|---|---|
| **Speed** | Milliseconds per message; no handshake after the first |
| **Memory** | A few KB per connection — 100k connections is real memory |
| **CPU** | Low per message, but many idle connections still cost |
| **Network** | Dramatically less than polling for frequent updates |

---

## 16. Security Considerations

> [!CAUTION]
> **The browser's same-origin policy does not apply to WebSockets.** Any website can open a WebSocket to your server. If you authenticate purely by cookie, a malicious page can connect as your logged-in user — this is **Cross-Site WebSocket Hijacking**.

- **Always validate the `Origin` header** on the server during the handshake
- **Always use `wss://`** — `ws://` is plaintext and will be blocked from HTTPS pages anyway
- **Authenticate during the handshake**, and re-check authorisation per message for sensitive actions
- **Rate-limit messages** — an open connection is an open pipe for abuse
- **Validate every incoming message**; a long-lived connection does not make its content trustworthy
- **Cap connections per user** to prevent resource exhaustion

---

## 17. Mental Model

> [!NOTE]
> **HTTP is sending letters. A WebSocket is an open phone line.**
>
> With letters, only the sender starts a conversation and each one costs an envelope and a stamp. With the line open, either side can speak the moment they have something to say — but somebody has to keep paying for the line, and the operator must remember which caller is on which line.

---

## 18. Mini Architecture Diagram

```text
Browser A          Browser B
    │  wss             │  wss
    ↓                  ↓
Server 1           Server 2
    └──────┬───────────┘
      Redis Pub/Sub        ← required as soon as there is more than one server
           ↓
        Database
```

---

## 19. Complete Request Flow

A chat message in a multi-server deployment:

```text
User A types a message
    ↓
Sent over the existing WebSocket to Server 1     (~1 ms, no handshake)
    ↓
Server 1 validates and authorises it
    ↓
Message persisted to the database
    ↓
Published to Redis channel "room:42"
    ↓
Server 2 is subscribed → receives it
    ↓
Server 2 pushes it down User B's WebSocket
    ↓
User B sees it — total latency in the tens of milliseconds
```

---

## 20. Key Takeaway

> [!IMPORTANT]
> WebSockets let the server push data over a persistent connection — and in exchange they make your servers stateful, which is why scaling them requires a shared pub/sub layer.

---

## 21. Common Mistakes

- **Not validating the `Origin` header** — cross-site WebSocket hijacking
- **Assuming it scales like HTTP** — it does not; you need pub/sub across instances
- **No reconnection logic** — connections drop constantly on mobile networks
- **No heartbeat/ping** — dead connections linger invisibly until you try to write
- **Using WebSockets for rare updates**, where polling or SSE is simpler
- **Skipping per-message authorisation** because the connection was authenticated once
- **Forgetting load balancer configuration** — many need explicit WebSocket support

---

## 22. Open Source Technologies

- **Socket.IO** — WebSockets with fallbacks, reconnection and rooms built in
- **ws**, **µWebSockets** — lower-level Node.js implementations
- **Phoenix Channels** (Elixir) — one of the best implementations of this model anywhere
- **Centrifugo**, **Soketi** — dedicated real-time servers you run alongside your API
- **Redis Pub/Sub**, **NATS** — the message backbone for multi-instance deployments

---

## 23. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 24. Workbook Exercise

- [ ] Build a minimal chat server and open it in two browser windows.
- [ ] Kill the server while connected and observe what the client does — then add reconnection with backoff.
- [ ] Explain in two sentences why a second server instance breaks a naive WebSocket chat.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
Client
    ↓  wss
WebSocket server
    ↓
Redis Pub/Sub  (fan-out across instances)
    ↓
Other WebSocket servers → other clients
```

## 2. Request Flow

```text
Input       an HTTP upgrade request, then messages from either side
    ↓
Processing  framed, validated, authorised, broadcast via pub/sub
    ↓
Output      messages pushed to connected clients in milliseconds
```

## 3. Real-World Usage

**Figma** synchronises multi-user design sessions over WebSockets. Every cursor movement and edit is an event pushed to every other participant — an interaction model that polling could not deliver at acceptable latency or cost.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | A persistent, bidirectional connection over TCP, started via HTTP |
| **Why does it exist?** | Because HTTP servers cannot initiate communication |
| **Where does it belong?** | The application layer, alongside HTTP, above TLS |
| **When should I use it?** | Frequent, low-latency, two-way updates — not occasional notifications |
