# TCP/IP

> **In one line —** IP gets a packet to the right machine; TCP makes sure the whole message arrives, in order, without duplicates.

| | |
|---|---|
| **Category** | Networking Protocol |
| **Architectural Layer** | Transport (TCP) + Internet (IP) |
| **Alternative** | UDP — fast, unreliable |
| **Related notes** | [IP Addresses](IP%20Addresses.md) · [Internet Fundamentals](Internet%20Fundamentals.md) · [HTTP HTTPS](HTTP%20HTTPS.md) · [WebSockets](WebSockets.md) · [System Calls](../02%20-%20Computer%20Science%20Fundamentals/System%20Calls.md) |

---

## 1. Short Definition

*What is it?*

**IP** delivers individual packets to an address, with no guarantee they arrive at all. **TCP** sits on top and turns that unreliable delivery into a reliable, ordered byte stream between two programs.

---

## 2. Purpose

*What is its main purpose?*

To let two applications exchange data as though they were connected by a private, reliable pipe — even though the underlying network loses, duplicates and reorders packets constantly.

---

## 3. Problem

*What engineering problem does it solve?*

The internet is deliberately unreliable. Routers drop packets when congested, paths change mid-conversation, and packets arrive out of order.

```text
Sent:      [1] [2] [3] [4] [5]
Arrived:   [1] [3] [5] [2]        ← 4 lost, order scrambled
    ↓  TCP
Delivered: [1] [2] [3] [4] [5]    ← 4 re-requested, order restored
```

Without TCP, every application would have to solve this itself.

---

## 4. Architecture Position

```text
Application     HTTP, WebSocket, SMTP        "what the bytes mean"
    ↓
TRANSPORT       TCP or UDP                   "which program, and did it arrive?"
    ↓
INTERNET        IP                           "which machine, and which way?"
    ↓
Link            Ethernet, Wi-Fi              "next physical hop"
```

Your code touches this through **sockets** — `connect`, `send`, `recv` are [system calls](../02%20-%20Computer%20Science%20Fundamentals/System%20Calls.md) into the kernel's TCP implementation.

---

## 5. Ports — how the machine knows which program

An IP address identifies a machine; a **port** identifies a program on it.

```text
93.184.216.34:443    →  the web server
93.184.216.34:5432   →  PostgreSQL
93.184.216.34:6379   →  Redis
```

| Range | Meaning |
|---|---|
| 0–1023 | Well-known (80 HTTP, 443 HTTPS, 22 SSH, 53 DNS) |
| 1024–49151 | Registered (5432 Postgres, 3306 MySQL, 6379 Redis) |
| 49152–65535 | Ephemeral — assigned to outgoing client connections |

---

## 6. The three-way handshake

Before any data flows, TCP establishes a connection:

```text
Client                          Server
   │  ──── SYN ──────────────►    │      "I want to connect"
   │  ◄─── SYN-ACK ───────────    │      "OK, and I want to as well"
   │  ──── ACK ──────────────►    │      "Confirmed"
   │                              │
   │  ══════ data flows ══════    │
```

> [!IMPORTANT]
> That handshake costs **one full round trip** before a single byte of your data moves. Across an ocean that is ~150 ms of pure setup — which is exactly why connection reuse and connection pooling matter so much.

---

## 7. Processing — how reliability works

```text
Every byte is numbered  (sequence numbers)
    ↓
Receiver acknowledges what it has received  (ACK)
    ↓
No ACK within the timeout?  →  retransmit
    ↓
Out-of-order arrivals are buffered and reordered
    ↓
Duplicates are discarded
```

TCP also manages **flow control** (do not send faster than the receiver can consume) and **congestion control** (slow down when the network starts dropping packets — this is what makes the internet stable under load).

---

## 8. TCP vs UDP

| | TCP | UDP |
|---|---|---|
| **Reliability** | Guaranteed delivery | None |
| **Order** | Guaranteed | None |
| **Connection** | Handshake required | None — just send |
| **Speed** | Slower (ACKs, retransmits) | Faster |
| **Overhead** | 20+ byte header | 8 byte header |
| **Used by** | HTTP, SSH, databases, email | DNS, video calls, gaming, QUIC |

> [!TIP]
> UDP is not "worse" — it is the right choice when **late data is useless**. In a video call, retransmitting a frame from two seconds ago is pointless; skipping it is correct.

---

## 9. Real World Example

- **Every database connection** is a TCP connection, which is why connection **pooling** exists — the handshake plus authentication is far too expensive to repeat per query.
- **HTTP keep-alive** reuses one TCP connection for many requests, for the same reason.
- **HTTP/3 abandoned TCP entirely** for QUIC over UDP, because TCP's head-of-line blocking meant one lost packet stalled every stream sharing the connection.

---

## 10. Communication and Dependencies

- **Applications** above, through the socket API
- **[IP](IP%20Addresses.md)** below, for routing
- **The kernel** — TCP is implemented in the [kernel](../02%20-%20Computer%20Science%20Fundamentals/Kernel.md), not in your process
- Requires a routable path and open ports through every firewall in between

---

## 11. When To Use

> [!TIP]
> **TCP** whenever correctness matters and every byte must arrive: APIs, databases, file transfer, messaging.
> **UDP** when timeliness beats completeness: real-time media, telemetry, DNS, gaming.

---

## 12. When NOT To Use

> [!CAUTION]
> - **Do not open a new TCP connection per operation** — pool them. This is one of the most common and most expensive performance mistakes in backend code.
> - **Do not use TCP for real-time media** — retransmitting stale audio creates a growing delay rather than fixing anything.
> - **Do not assume a connection is alive** because it was a minute ago. Firewalls and NAT devices silently drop idle connections; configure keep-alives and timeouts.

---

## 13. Advantages and Disadvantages

**TCP advantages**
- Reliable, ordered, duplicate-free delivery
- Flow and congestion control built in
- Universally supported and battle-tested for decades

**TCP disadvantages**
- Handshake latency before any data
- **Head-of-line blocking** — one lost packet stalls everything behind it
- Higher overhead per byte
- Connection state must be held on both ends, limiting how many can be open

---

## 14. Performance Impact

| Resource | Impact |
|---|---|
| **Speed** | +1 round trip to connect; +2 more if TLS follows |
| **Memory** | Kernel buffers per connection — a real limit at high connection counts |
| **CPU** | Modest; offloaded to the network card on servers |
| **Network** | Retransmissions and ACKs add traffic under packet loss |

> [!IMPORTANT]
> On a lossy connection, TCP throughput collapses far more than the loss rate alone suggests, because congestion control interprets loss as congestion and backs off.

---

## 15. Security Considerations

> [!CAUTION]
> **TCP provides no encryption and no authentication.** Everything is plaintext on the wire unless [TLS](TLS%20SSL.md) is layered on top — which is precisely why HTTPS exists.

- **SYN flood** — an attacker sends handshake requests it never completes, exhausting connection state. SYN cookies mitigate it.
- **Connection hijacking** was historically possible with predictable sequence numbers; modern stacks randomise them.
- **Port scanning** reveals what is running; only expose the ports you need.
- **Idle connections held open** are a resource-exhaustion vector — always set timeouts.

---

## 16. Mental Model

> [!NOTE]
> **TCP is registered post with delivery confirmation.**
>
> Every parcel is numbered; the recipient signs for each one; anything unconfirmed is sent again; and the sender slows down if the depot looks overloaded. **UDP is dropping a postcard in the box** — cheap, immediate, and you will never know if it arrived.

---

## 17. Mini Architecture Diagram

```text
Your application
    ↓  socket API (send / recv)
┌──── TCP (in the kernel) ────┐
│  sequence numbers, ACKs      │
│  retransmission, reordering  │
│  flow + congestion control   │
└──────────────┬───────────────┘
               ↓
              IP  → routing across the internet
               ↓
           Ethernet / Wi-Fi
```

---

## 18. Complete Request Flow

One HTTPS request, from nothing to bytes:

```text
DNS resolves the hostname                          ~20–100 ms
    ↓
TCP three-way handshake                            1 round trip
    ↓
TLS handshake                                      1–2 round trips
    ↓
HTTP request sent
    ↓
Server responds; TCP acknowledges each segment
    ↓
Any lost segment is retransmitted automatically
    ↓
Connection kept alive for the next request         ← this is the saving
```

---

## 19. Key Takeaway

> [!IMPORTANT]
> TCP turns an unreliable packet network into a reliable byte stream, and pays for it with a setup round trip — so reuse connections instead of creating them.

---

## 20. Common Mistakes

- **Not pooling connections** to databases and external services
- **Disabling HTTP keep-alive** without realising the cost
- **Ignoring timeouts** — a connection with no timeout can hang a thread indefinitely
- **Assuming TCP is secure** — it is not encrypted or authenticated
- **Confusing "connected" with "working"** — a half-open connection looks alive until the first write fails
- **Exhausting ephemeral ports** by opening thousands of short-lived connections

---

## 21. Open Source Technologies

- **Linux kernel TCP stack** — the reference implementation most of the internet runs on
- **QUIC / HTTP/3** — TCP's successor for the web, built on UDP
- **tcpdump**, **Wireshark**, **ss**, **netstat** — inspect connections and packets
- **HAProxy**, **Nginx** — TCP and HTTP load balancing

---

## 22. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 23. Workbook Exercise

- [ ] Capture a TCP handshake with Wireshark or `tcpdump` and identify SYN, SYN-ACK and ACK.
- [ ] Check whether your application pools database connections, and how large the pool is.
- [ ] Explain in two sentences why HTTP/3 moved from TCP to UDP.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
Application (HTTP)
    ↓
TCP  (reliability, ordering, ports)
    ↓
IP   (addressing, routing)
    ↓
Ethernet / Wi-Fi
```

## 2. Request Flow

```text
Input       a stream of bytes from an application
    ↓
Processing  segmented, numbered, acknowledged, retransmitted, reordered
    ↓
Output      the same byte stream, intact and in order, at the other end
```

## 3. Real-World Usage

**HTTP/3** exists because TCP's head-of-line blocking hurt page loads: one lost packet stalled every parallel stream on the connection. Google built QUIC on UDP and reimplemented reliability per stream — a rare case of the industry replacing a foundational protocol.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | IP routes packets; TCP makes the byte stream reliable and ordered |
| **Why does it exist?** | Because packet networks lose, duplicate and reorder data |
| **Where does it belong?** | Between applications and IP routing, implemented in the kernel |
| **When should I use it?** | TCP when every byte matters; UDP when timeliness matters more |
