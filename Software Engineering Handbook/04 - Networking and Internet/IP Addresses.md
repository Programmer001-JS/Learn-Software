# IP Addresses

> **In one line —** the numeric address of a machine on a network; without it, a packet has no idea where to go.

| | |
|---|---|
| **Category** | Networking Concept |
| **Architectural Layer** | Internet (Layer 3) |
| **Versions** | IPv4 (32-bit) · IPv6 (128-bit) |
| **Related notes** | [Internet Fundamentals](Internet%20Fundamentals.md) · [DNS](DNS.md) · [TCP IP](TCP%20IP.md) · [ISP Router Switch Modem](ISP%20Router%20Switch%20Modem.md) · [VPC](../12%20-%20Cloud%20Architecture/VPC.md) |

---

## 1. Short Definition

*What is it?*

An IP address identifies a network interface so that packets can be routed to it. **IPv4** looks like `192.168.1.10` (four numbers, 0–255). **IPv6** looks like `2001:0db8::8a2e:0370:7334` and exists because IPv4 addresses ran out.

---

## 2. Purpose

*What is its main purpose?*

To give every reachable machine a globally meaningful destination, so routers can decide which direction to forward each packet without knowing the whole path.

---

## 3. Problem

*What engineering problem does it solve?*

Routing needs a hierarchical, machine-readable address. Names are for humans ([DNS](DNS.md) handles those); MAC addresses only work within one local network. IP is the layer that works globally.

---

## 4. Architecture Position

```text
Application    example.com                      ← humans
    ↓  DNS
IP address     93.184.216.34                    ← routing  (you are here)
    ↓  ARP
MAC address    00:1A:2B:3C:4D:5E                ← local delivery
    ↓
Physical wire
```

---

## 5. Public vs private addresses

```text
PRIVATE — not routable on the internet, reusable by everyone
    10.0.0.0    – 10.255.255.255
    172.16.0.0  – 172.31.255.255
    192.168.0.0 – 192.168.255.255

PUBLIC — globally unique, routable
    everything else
```

> [!IMPORTANT]
> Your laptop's `192.168.1.7` is not unique — millions of devices have it. **NAT** on your router translates between it and one shared public address, which is why unsolicited inbound connections do not reach you.

---

## 6. IPv4 vs IPv6

| | IPv4 | IPv6 |
|---|---|---|
| **Size** | 32 bits | 128 bits |
| **Total addresses** | ~4.3 billion | ~340 undecillion |
| **Notation** | `93.184.216.34` | `2001:db8::1` |
| **NAT** | Ubiquitous, out of necessity | Rarely needed |
| **Status** | Exhausted, still dominant | Growing, roughly half of traffic |

IPv4 ran out because 4.3 billion addresses seemed limitless in 1981. NAT postponed the crisis for decades and, in doing so, slowed IPv6 adoption considerably.

---

## 7. Subnets and CIDR

CIDR notation splits an address into a **network part** and a **host part**.

```text
192.168.1.0/24
              ↑
        first 24 bits identify the network
        remaining 8 bits identify the host  → 256 addresses

/24  →  256 addresses      typical small subnet
/16  →  65,536 addresses   typical VPC
/32  →  1 address          a single host
```

> [!TIP]
> The smaller the number after the slash, the bigger the network. This matters daily in cloud work — [VPC](../12%20-%20Cloud%20Architecture/VPC.md) subnets, security group rules and firewall entries are all written in CIDR.

---

## 8. Real World Example

- **Cloud VPCs** — you choose a CIDR block such as `10.0.0.0/16` and carve it into subnets. Choosing overlapping ranges across environments causes painful problems when you later need to connect them.
- **Firewall and security group rules** are written as CIDR ranges.
- **Rate limiting and geo-blocking** work on IP, which is why VPNs defeat both easily.
- **`X-Forwarded-For`** — behind a load balancer or CDN, the connecting IP is the proxy's; the real client IP arrives in a header.

---

## 9. Input, Processing, Output

**Input:** a packet with a destination IP address.

**Processing:** each router compares the destination against its routing table, finds the most specific matching prefix, and forwards to the next hop.

**Output:** the packet arrives at the destination interface — or is dropped and, at best, an ICMP error is returned.

---

## 10. Communication

- **[DNS](DNS.md)** — turns names into IP addresses
- **ARP** — turns an IP into a MAC address on the local network
- **Routers** — forward based on it
- **Firewalls and security groups** — filter based on it

---

## 11. Alternatives

There is no alternative to IP for internet routing. What varies is how addresses are assigned and abstracted:

```text
Static IP        fixed, used for servers
DHCP             assigned automatically, used for clients
NAT              many private addresses behind one public one
Service mesh     services addressed by name; IPs become an implementation detail
```

---

## 12. When To Use

> [!TIP]
> Address by **name**, not by IP, wherever possible. Hard-coded IP addresses break the moment a server is replaced — which in cloud and container environments is constantly.

---

## 13. When NOT To Use

> [!CAUTION]
> - **Do not use IP as identity.** Users share IPs behind NAT and change them on mobile networks. IP-based authentication is not authentication.
> - **Do not hard-code IPs** in configuration; use DNS names or service discovery.
> - **Do not rely on GeoIP for anything security-critical** — VPNs make it a hint, not a fact.

---

## 14. Advantages and Disadvantages

**Advantages**
- Hierarchical, so routing tables stay manageable
- Works identically for every application above it
- Well-understood filtering and access control

**Disadvantages**
- IPv4 exhaustion forced NAT, which broke end-to-end connectivity
- Addresses change; they are a poor identifier for anything long-lived
- IPv6 adoption has been slow and dual-stack adds operational complexity

---

## 15. Performance Impact

| Resource | Impact |
|---|---|
| **Speed** | Routing lookups are hardware-fast; the physical distance dominates |
| **Memory** | Backbone routing tables hold ~1 million routes |
| **CPU** | Negligible in normal applications |
| **Network** | IPv6 headers are larger but simpler to process |

---

## 16. Security Considerations

> [!CAUTION]
> **IP addresses are trivially spoofed** in UDP-based protocols. This is what makes reflection and amplification DDoS attacks possible, and it is why UDP services must be designed defensively.

- **`X-Forwarded-For` can be forged** unless your proxy is configured to overwrite it — trusting it blindly bypasses IP-based rate limiting
- **IP allow-lists** are a useful additional layer, never a primary authentication mechanism
- **Logging IP addresses is processing personal data** under GDPR; be deliberate about retention
- **SSRF** attacks abuse private ranges — validate that user-supplied URLs do not resolve to `169.254.169.254` or internal subnets

---

## 17. Mental Model

> [!NOTE]
> **An IP address is a street address; a MAC address is the name on the door.**
>
> The postal system routes by address across the whole country. Only once the parcel reaches the right building does the name on the door matter. And behind NAT, an entire office block shares one street address, with a receptionist deciding which desk each delivery belongs to.

---

## 18. Mini Architecture Diagram

```text
example.com
    ↓  DNS
93.184.216.34
    ↓  routing table lookup at each hop
Router → Router → Router
    ↓  ARP on the final network
MAC address → the physical machine
```

---

## 19. Complete Request Flow

```text
Application wants example.com
    ↓
DNS resolves it to 93.184.216.34
    ↓
Is that address inside my subnet?
    ↓ NO → send to the default gateway (the router)
    ↓
Router checks its table → forwards toward the destination
    ↓
...repeated at each hop across the internet
    ↓
Final router ARPs for the MAC address on the local network
    ↓
Packet delivered to the server's network card
```

---

## 20. Key Takeaway

> [!IMPORTANT]
> An IP address is where a machine is, not who it is — it identifies a location on the network, changes frequently, and must never be used as identity.

---

## 21. Common Mistakes

- **Hard-coding IP addresses** instead of using DNS or service discovery
- **Using IP as user identity** or as an authentication factor
- **Trusting `X-Forwarded-For`** without a properly configured proxy
- **Overlapping CIDR ranges** across environments, which blocks future VPC peering
- **Forgetting IPv6** — a service reachable only over IPv4 is invisible to some users
- **Confusing public and private addressing** when debugging "it works on my machine"

---

## 22. Open Source Technologies

- **iproute2**, **dig**, **traceroute**, **mtr** — inspection and diagnosis
- **iptables**, **nftables** — filtering by address
- **CoreDNS**, **Consul** — service discovery replacing hard-coded addresses
- **WireGuard** — modern VPN, useful for private inter-network routing

---

## 23. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 24. Workbook Exercise

- [ ] Find your private and public IP, and explain the difference in two sentences.
- [ ] Work out how many usable addresses a `/22` subnet contains.
- [ ] Check whether your application correctly identifies the real client IP behind a proxy.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
Hostname
    ↓  DNS
IP address
    ↓  routing
Network interface
    ↓  ARP
MAC address → machine
```

## 2. Request Flow

```text
Input       a packet with a destination IP
    ↓
Processing  longest-prefix match at each router; forwarded hop by hop
    ↓
Output      delivery to the destination interface, or a drop
```

## 3. Real-World Usage

Every **cloud VPC** starts with choosing a CIDR block. Teams that pick `10.0.0.0/16` for every environment discover the cost years later, when two environments must be connected and their address ranges collide.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | The numeric address identifying a network interface |
| **Why does it exist?** | Because routers need a hierarchical, machine-readable destination |
| **Where does it belong?** | Layer 3, between DNS names above and MAC addresses below |
| **When should I use it?** | For routing and filtering — never as identity |
