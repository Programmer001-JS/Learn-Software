# DNS

> **In one line —** the phone book of the internet: it turns a name people can remember into an address routers can use.

| | |
|---|---|
| **Full name** | Domain Name System |
| **Category** | Networking Infrastructure |
| **Architectural Layer** | Application (but foundational to everything) |
| **Related notes** | [IP Addresses](IP%20Addresses.md) · [Internet Fundamentals](Internet%20Fundamentals.md) · [HTTP HTTPS](HTTP%20HTTPS.md) · [Load Balancing](../14%20-%20Scalability%20and%20Reliability/Load%20Balancing.md) |

---

## 1. Short Definition

*What is it?*

DNS is a globally distributed database that maps domain names to [IP addresses](IP%20Addresses.md) and other records. Every connection to a named host begins with a DNS lookup.

---

## 2. Purpose

*What is its main purpose?*

To decouple **names from addresses**. You can move a service to an entirely different server, provider or continent, and users keep typing the same name.

---

## 3. Problem

*What engineering problem does it solve?*

Humans cannot remember `93.184.216.34`, and addresses change constantly — servers are replaced, providers change, load is redistributed. Hard-coding addresses anywhere makes all of that impossible.

---

## 4. Architecture Position

```text
User types example.com
    ↓
DNS RESOLUTION            ← you are here; happens before anything else
    ↓
IP address
    ↓
TCP connection → TLS → HTTP
```

> [!IMPORTANT]
> DNS is the **first** thing that happens and a single point of failure for everything after it. A large share of major internet outages have been DNS problems, not server problems.

---

## 5. Processing — how resolution works

```text
Browser cache?                    → hit → done
    ↓ miss
Operating system cache?           → hit → done
    ↓ miss
Recursive resolver (ISP, 1.1.1.1, 8.8.8.8)
    ↓ cached? → done
    ↓
ROOT servers        "who handles .com?"
    ↓
TLD servers (.com)  "who handles example.com?"
    ↓
Authoritative server for example.com  →  93.184.216.34
    ↓
Cached at every layer for the TTL, then returned
```

The hierarchy is why one system can serve the entire internet: no server knows everything, each only knows who to ask next.

---

## 6. Record types worth knowing

| Record | Purpose |
|---|---|
| **A** | Name → IPv4 address |
| **AAAA** | Name → IPv6 address |
| **CNAME** | Name → another name (alias) |
| **MX** | Where email for this domain goes |
| **TXT** | Arbitrary text — used for domain verification, SPF, DKIM |
| **NS** | Which servers are authoritative for this domain |
| **SRV** | Service discovery — host and port |

---

## 7. TTL — the setting that matters most operationally

Every record carries a **time to live**, telling resolvers how long they may cache it.

```text
TTL 300     (5 min)   changes propagate quickly, more DNS traffic
TTL 86400   (24 h)    efficient, but a migration takes a day to take effect
```

> [!TIP]
> **Lower the TTL a day or two before a planned migration**, then raise it afterwards. Changing a record with a 24-hour TTL means some users keep reaching the old server for a full day.

---

## 8. Real World Example

- **Blue-green deployments and failover** — switch traffic by changing a DNS record, subject to TTL.
- **CDNs** return *different* IP addresses depending on where the query came from, sending each user to the nearest edge.
- **Kubernetes** runs CoreDNS internally so services address each other by name rather than by pod IP.
- **Email deliverability** depends entirely on TXT records — SPF, DKIM and DMARC.

---

## 9. Input and Output

**Input:** a domain name and a record type.
**Output:** the record value (usually an IP address) plus a TTL, or `NXDOMAIN` if the name does not exist.

---

## 10. Communication

- **Resolvers** — recursive lookup on your behalf, and cache aggressively
- **Authoritative servers** — hold the real answers for a domain
- **Registrars** — where you buy a domain and set its NS records
- **Every client** — browsers, servers, containers all resolve constantly

---

## 11. Dependencies

- A configured resolver (from DHCP, `/etc/resolv.conf`, or explicit configuration)
- Working network connectivity — DNS uses UDP port 53, with TCP as a fallback
- Correctly delegated NS records at the registrar

---

## 12. Alternatives

```text
DNS                    the universal standard
    ↓
/etc/hosts             local override — useful for testing, unmanageable at scale
    ↓
Service discovery      Consul, etcd, Kubernetes DNS — for internal services
    ↓
Service mesh           name-based routing with health awareness built in
```

---

## 13. When To Use

> [!TIP]
> Always address services by name. DNS gives you the freedom to change infrastructure without changing clients — that indirection is the entire point.

---

## 14. When NOT To Use

> [!CAUTION]
> **Do not rely on DNS for fast failover.** Caching means a change can take minutes to hours to reach everyone, regardless of your TTL, because some resolvers ignore short TTLs. For fast failover use a [load balancer](../14%20-%20Scalability%20and%20Reliability/Load%20Balancing.md) with health checks; use DNS for slower, planned changes.

---

## 15. Advantages

- Decouples names from addresses completely
- Hierarchical and distributed — no central bottleneck
- Caching makes the vast majority of lookups instant
- Supports geographic and weighted routing

---

## 16. Disadvantages

- Caching makes changes slow and unpredictable to propagate
- A single point of failure for everything downstream
- Plaintext by default — resolvers and networks see every name you look up
- Misconfiguration is easy and the failure modes are confusing

---

## 17. Performance Impact

| Resource | Impact |
|---|---|
| **Speed** | 20–120 ms uncached; ~0 ms cached |
| **Memory** | Negligible |
| **CPU** | Negligible |
| **Network** | One extra round trip before the connection can even begin |

> [!TIP]
> Page load time often includes several DNS lookups, one per distinct host. Reducing the number of third-party domains a page contacts is a real and frequently overlooked performance win.

---

## 18. Security Considerations

> [!CAUTION]
> DNS is unauthenticated and unencrypted by default. **DNS spoofing** and **cache poisoning** let an attacker send users to the wrong server entirely — with no visible symptom until the certificate check fails.

- **DNSSEC** signs records cryptographically, proving they were not tampered with
- **DoH / DoT** (DNS over HTTPS / TLS) encrypt queries so the network cannot read or modify them
- **Subdomain takeover** — a CNAME pointing at a decommissioned cloud resource lets someone else claim that hostname; audit dangling records
- **DNS exfiltration** — attackers tunnel stolen data out through DNS queries, because port 53 is almost never blocked
- **Registrar account security** is critical: losing control of a domain is losing everything attached to it. Use strong MFA and registrar lock.

---

## 19. Mental Model

> [!NOTE]
> **DNS is the phone book, and every phone keeps its own copy of the pages it used recently.**
>
> Changing your number is easy. Getting everyone to notice takes as long as their old copy stays useful — which is exactly what TTL controls.

---

## 20. Mini Architecture Diagram

```text
Client
    ↓
Local cache → OS cache → Recursive resolver
                              ↓
                        Root servers
                              ↓
                        TLD (.com) servers
                              ↓
                    Authoritative server
                              ↓
                        IP address
```

---

## 21. Complete Request Flow

Opening `https://example.com`:

```text
Browser checks its own cache            → miss
    ↓
OS resolver cache                       → miss
    ↓
Recursive resolver (e.g. 1.1.1.1)       → miss
    ↓
Root  → ".com is handled by these servers"
    ↓
.com TLD → "example.com is handled by ns1.example.com"
    ↓
Authoritative → "A record: 93.184.216.34, TTL 300"
    ↓
Answer cached at every level for 300 seconds
    ↓
Browser opens a TCP connection to 93.184.216.34
```

---

## 22. Key Takeaway

> [!IMPORTANT]
> DNS turns names into addresses and is the first step of every connection — its caching makes it fast and makes changes slow, and it is a single point of failure for everything above it.

---

## 23. Common Mistakes

- **Changing a record without lowering the TTL first**
- **Using DNS for fast failover** and being surprised by cached answers
- **Forgetting DNS in incident response** — "the site is down" is frequently DNS
- **Leaving dangling CNAMEs** pointing at deleted cloud resources — a subdomain takeover risk
- **Not securing the registrar account**, which is the true root of your domain's security
- **Ignoring propagation** when testing — your own machine may be cached differently from everyone else's

---

## 24. Open Source Technologies

- **BIND**, **PowerDNS**, **Knot** — authoritative DNS servers
- **Unbound**, **dnsmasq** — recursive resolvers
- **CoreDNS** — the DNS server inside Kubernetes
- **dig**, **nslookup**, **dnsviz** — inspection and debugging

---

## 25. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 26. Workbook Exercise

- [ ] Run `dig example.com` and identify the record type, value and TTL in the output.
- [ ] Check the TTL on your own domain's records and decide whether it suits your deployment process.
- [ ] Audit your DNS records for any CNAME pointing at a resource you no longer control.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
Name
    ↓
DNS (cached at every layer)
    ↓
IP address
    ↓
TCP → TLS → HTTP
```

## 2. Request Flow

```text
Input       a domain name and a record type
    ↓
Processing  cache checks, then a hierarchical walk from root to authoritative
    ↓
Output      an IP address plus a TTL
```

## 3. Real-World Usage

Several of the largest internet outages in recent years — affecting Facebook, Slack and others — were caused by DNS or routing misconfiguration rather than server failure. When DNS is wrong, perfectly healthy servers become unreachable.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | A distributed system mapping domain names to IP addresses |
| **Why does it exist?** | So that names stay stable while infrastructure changes underneath |
| **Where does it belong?** | The first step of every connection to a named host |
| **When should I use it?** | Always — but never for failover that must happen in seconds |
