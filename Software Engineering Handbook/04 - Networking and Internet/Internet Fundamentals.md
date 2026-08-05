# Internet Fundamentals

> **In one line —** the internet is not a network; it is millions of independent networks that agreed on one set of rules for passing packets to each other.

| | |
|---|---|
| **Category** | Overview note *(hub for this section)* |
| **Architectural Layer** | Network |
| **Related notes** | [IP Addresses](IP%20Addresses.md) · [DNS](DNS.md) · [TCP IP](TCP%20IP.md) · [HTTP HTTPS](HTTP%20HTTPS.md) · [TLS SSL](TLS%20SSL.md) · [ISP Router Switch Modem](ISP%20Router%20Switch%20Modem.md) |

---

## 1. Short Definition

*What is it?*

The internet is a global system of interconnected networks that all speak the **Internet Protocol**. There is no central computer and no owner — only an agreement on how to address and forward small chunks of data called packets.

---

## 2. Purpose

*What is its main purpose?*

To move data between any two machines in the world without either of them knowing the route in advance, and without any single organisation controlling the path.

---

## 3. Problem

*What engineering problem does it solve?*

Networks were originally islands: a university network could not talk to a corporate one. The internet's founding idea was that networks would remain independent and simply agree on a common envelope format.

> [!IMPORTANT]
> This is why the internet is resilient: no machine knows the whole route. Each router only knows the next hop, so failures are routed around automatically rather than reported to some central authority.

---

## 4. The layer model

Every layer treats the one below as a service and adds its own envelope.

```text
APPLICATION    HTTP, DNS, SMTP, WebSocket     "what does this data mean?"
    ↓
TRANSPORT      TCP, UDP                       "which program, and did it arrive?"
    ↓
INTERNET       IP                             "which machine, and which way?"
    ↓
LINK           Ethernet, Wi-Fi                "next hop on this physical wire"
```

| Layer | Adds | Identifier |
|---|---|---|
| Application | Meaning | URL, hostname |
| Transport | Reliability, ports | Port number |
| Internet | Global addressing | IP address |
| Link | Physical delivery | MAC address |

---

## 5. Packet switching

Data is not sent as a continuous stream. It is chopped into **packets**, each routed independently, possibly along different paths, then reassembled at the destination.

```text
Message
    ↓  split
[pkt 1] [pkt 2] [pkt 3] [pkt 4]
    ↓  each routed independently, hop by hop
Arrive out of order, some lost
    ↓  TCP reorders and re-requests the missing ones
Original message reconstructed
```

---

## 6. Real World Example

- **Opening a website** touches nearly every concept in this section: [DNS](DNS.md), [TCP](TCP%20IP.md), [TLS](TLS%20SSL.md), [HTTP](HTTP%20HTTPS.md), routing and often a CDN.
- **Undersea cables** carry the overwhelming majority of intercontinental traffic. This is why latency between continents cannot be optimised away — it is limited by the speed of light in glass.
- **CDNs** exist entirely because of that physical limit: bring the data closer rather than making the network faster.

---

## 7. Latency — the number that shapes architecture

```text
Same data centre          ~0.5 ms
Same city                 ~5 ms
Same continent            ~30 ms
Across an ocean           ~100–150 ms
Satellite (geostationary) ~600 ms
```

> [!IMPORTANT]
> A network round trip is roughly a **million times** slower than a RAM read. This single fact justifies caching, CDNs, batching, connection pooling, and the general rule that fewer, larger calls beat many small ones.

---

## 8. What actually happens when you open a website

```text
1. DNS       "what is the IP for example.com?"        ~20–100 ms
    ↓
2. TCP       three-way handshake with that IP          1 round trip
    ↓
3. TLS       certificate check, key exchange           1–2 round trips
    ↓
4. HTTP      GET /  → server responds
    ↓
5. Browser   parse HTML, discover CSS/JS/images
    ↓
6. Repeat 1–4 for every additional host involved
    ↓
7. Render
```

> [!TIP]
> Notice how many round trips happen before a single byte of content arrives. This is why HTTP/2 multiplexing, connection reuse and CDNs matter so much for perceived speed.

---

## 9. Mental Model

> [!NOTE]
> **The internet is the postal system.**
>
> [IP addresses](IP%20Addresses.md) are street addresses. [DNS](DNS.md) is the phone book that turns a name into an address. Routers are sorting offices, each knowing only where to send a parcel next. [TCP](TCP%20IP.md) is registered delivery with confirmation and re-sending; UDP is dropping a postcard in the box and hoping.

---

## 10. Key Takeaway

> [!IMPORTANT]
> The internet moves independently routed packets between cooperating networks — and because a network round trip is enormously expensive compared to local work, minimising round trips is one of the highest-leverage optimisations in any system.

---

## 11. Common Mistakes

- **Ignoring latency** in architecture — chatty service-to-service calls destroy performance
- **Assuming the network is reliable** — packets are lost, connections drop, DNS fails
- **Assuming bandwidth solves latency** — a wider road does not shorten the distance
- **Forgetting that everything unencrypted is readable** by every network in the path
- **Not accounting for geography** — users on another continent experience a different product

---

## 12. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 13. Workbook Exercise

- [ ] Run `traceroute` (or `tracert`) to a site on another continent and count the hops.
- [ ] Open your browser's network tab and identify DNS, connection, TLS and download time for one request.
- [ ] Count how many separate hosts one page of your own project contacts.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
Browser
    ↓
DNS → IP address
    ↓
TCP connection
    ↓
TLS encryption
    ↓
HTTP request
    ↓
Server
```

## 2. Request Flow

```text
Input       a hostname and a request
    ↓
Processing  name resolution, connection, encryption, packet routing hop by hop
    ↓
Output      a response reassembled from independently routed packets
```

## 3. Real-World Usage

**Cloudflare and other CDNs** exist because of physics: nothing can make a request to another continent faster than light through glass. Their entire business is placing a copy of the content nearer to the user.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | Independent networks cooperating through a shared protocol |
| **Why does it exist?** | To connect networks without central control or a fixed route |
| **Where does it belong?** | Between every client and every server you will ever build |
| **When should I use it?** | Always — and design to cross it as rarely as possible |
