# TLS / SSL

> **In one line —** the layer that turns a readable, modifiable connection into an encrypted one, and proves the server is who it claims to be.

| | |
|---|---|
| **Full name** | Transport Layer Security (formerly Secure Sockets Layer) |
| **Category** | Security Protocol |
| **Architectural Layer** | Between TCP and the application |
| **Current versions** | TLS 1.2, TLS 1.3 — everything older is broken |
| **Related notes** | [HTTP HTTPS](HTTP%20HTTPS.md) · [TCP IP](TCP%20IP.md) · [Authentication](../09%20-%20Security/Authentication.md) · [Secrets Management](../09%20-%20Security/Secrets%20Management.md) |

---

## 1. Short Definition

*What is it?*

TLS is a protocol that wraps a [TCP](TCP%20IP.md) connection in encryption. **SSL** is its obsolete predecessor — the name survives in habit only; every SSL version is broken and disabled.

---

## 2. Purpose

*What is its main purpose?*

Three guarantees at once:

- **Confidentiality** — nobody in the path can read the traffic
- **Integrity** — nobody can modify it undetected
- **Authenticity** — you are talking to the real server, not an impostor

---

## 3. Problem

*What engineering problem does it solve?*

Traffic crosses networks you do not control — coffee shop Wi-Fi, ISPs, national backbones, cloud providers. Without TLS, every one of them can read passwords and *rewrite responses*. This is not hypothetical; ISPs have injected advertisements into plain HTTP pages.

---

## 4. Architecture Position

```text
HTTP / gRPC / SMTP / WebSocket
    ↓
TLS               ← you are here
    ↓
TCP
    ↓
IP → network
```

TLS is deliberately application-agnostic: the same layer secures HTTPS, encrypted email, database connections and message queues.

---

## 5. The handshake

```text
Client                                   Server
   │ ── ClientHello ────────────────►      │   supported versions and ciphers
   │ ◄─ ServerHello + CERTIFICATE ──       │   chosen cipher + identity proof
   │                                       │
   │  [client verifies the certificate]    │
   │                                       │
   │ ── key exchange ──────────────►       │   agree on a shared secret
   │ ◄─ Finished ──────────────────        │
   │ ══════ encrypted from here ══════     │
```

> [!IMPORTANT]
> **TLS 1.3 completes this in one round trip** (down from two in TLS 1.2), and supports 0-RTT resumption for repeat visitors. On a connection to another continent that is ~150 ms saved per connection — a genuine, user-visible improvement.

---

## 6. Certificates and the chain of trust

A certificate binds a **domain name** to a **public key**, signed by a Certificate Authority.

```text
Root CA           pre-installed in your OS and browser — trusted implicitly
    ↓ signs
Intermediate CA
    ↓ signs
example.com certificate
```

The browser verifies: is the signature chain valid, does the name match, has it expired, has it been revoked?

> [!CAUTION]
> The entire model rests on trusting hundreds of root CAs. A compromised or coerced CA can issue a valid certificate for any domain — which is why **Certificate Transparency** logs exist, so wrongly issued certificates become publicly visible.

---

## 7. Encryption — the two kinds, and why both

```text
ASYMMETRIC  (RSA, ECDSA)     slow, but no shared secret needed in advance
    ↓  used ONLY during the handshake to agree on a key
SYMMETRIC   (AES, ChaCha20)  fast, used for all the actual data
```

> [!TIP]
> This is the central trick of TLS: use expensive asymmetric cryptography once, to safely agree on a cheap symmetric key, then use that for everything else.

---

## 8. Real World Example

- **Let's Encrypt** made certificates free and automated, which is the single biggest reason HTTPS went from a minority to near-universal.
- **Browsers now mark plain HTTP as "Not Secure"**, and block features like geolocation and service workers on it.
- **TLS termination at the load balancer** is the standard pattern: the proxy decrypts, then talks plain HTTP to backends inside a private network.
- **Certificate expiry** is a recurring cause of production outages — a certificate silently expires and every client refuses to connect at once.

---

## 9. Input, Processing, Output

**Input:** a TCP connection, a server certificate and private key, and a list of acceptable cipher suites.

**Processing:** negotiate, authenticate, derive a shared symmetric key, then encrypt and authenticate every record.

**Output:** an encrypted, tamper-evident byte stream that behaves exactly like a normal socket to the application above.

---

## 10. Communication and Dependencies

- **Applications** above it — unchanged, they just open a TLS socket
- **[TCP](TCP%20IP.md)** below it
- A **certificate and private key** on the server
- A **trust store** of root CAs on the client
- **Accurate system time** — clock skew causes spurious "certificate not yet valid" errors

---

## 11. Alternatives

```text
TLS              the universal standard
    ↓
mTLS             both sides present certificates — standard inside service meshes
    ↓
WireGuard / IPsec  encrypt at the network layer instead, for whole networks
    ↓
Application-level encryption   end-to-end, so even the server cannot read the content
```

---

## 12. When To Use

> [!TIP]
> Always, for anything crossing a network you do not fully control — and increasingly inside private networks too. Zero-trust architecture assumes the internal network is hostile, which is why **mTLS between internal services** has become normal.

---

## 13. When NOT To Use

> [!CAUTION]
> The only reasonable exceptions are traffic on a genuinely isolated local socket, or a proxy that has already terminated TLS talking to a backend over a private network. Even then, encrypting internal traffic is increasingly the default rather than the exception.
>
> **Never** disable certificate verification to "make it work". That single line — `verify=False`, `rejectUnauthorized: false`, `-k` — removes the entire authenticity guarantee and turns HTTPS into slower HTTP.

---

## 14. Advantages and Disadvantages

**Advantages**
- Confidentiality, integrity and authenticity together
- Transparent to the application above
- Free, automated certificates make it effectively costless
- TLS 1.3 removed most historically weak options by design

**Disadvantages**
- Adds a round trip to connection setup (one with TLS 1.3)
- Certificates expire and must be renewed automatically
- Some CPU cost, though modern hardware makes it small
- Makes network debugging harder — you cannot read your own traffic without effort

---

## 15. Performance Impact

| Resource | Impact |
|---|---|
| **Speed** | +1 round trip (TLS 1.3) or +2 (TLS 1.2); ~0 on a resumed session |
| **Memory** | Session state per connection |
| **CPU** | A few percent; hardware AES makes bulk encryption nearly free |
| **Network** | Slight overhead per record |

> [!TIP]
> "TLS is too slow" has not been true for a decade. Session resumption, TLS 1.3 and hardware acceleration have made the cost negligible compared with a single database query.

---

## 16. Security Considerations

> [!CAUTION]
> **Certificate expiry is one of the most common self-inflicted outages in the industry.** Automate renewal, and monitor expiry dates independently of the renewal system — because the renewal system is exactly what fails.

- **Disable TLS 1.0/1.1 and all SSL versions** — they are broken, not merely old
- **HSTS** tells browsers never to use plaintext for your domain again
- **Never disable certificate verification** in clients, including in tests that later become production code
- **Private keys are the crown jewels** — keep them out of git, out of images, and in a secrets manager
- **mTLS** for internal service-to-service authentication
- **Certificate Transparency monitoring** alerts you if someone issues a certificate for your domain

---

## 17. Mental Model

> [!NOTE]
> **TLS is a sealed envelope with a notarised signature.**
>
> The envelope means nobody along the way can read the letter. The tamper-evident seal means you would notice if they had. The notary's signature — the certificate — proves the letter really came from the company it claims to be from, and not from someone who intercepted the post.

---

## 18. Mini Architecture Diagram

```text
Client                                    Server
  │                                          │
  │  TCP handshake                           │
  │  TLS handshake ── certificate ──────►    │
  │  verify chain, name, expiry              │
  │  derive shared symmetric key             │
  │                                          │
  │  ══ encrypted application data ══        │
  │                                          │
      ↓
  Load balancer terminates TLS
      ↓
  plain HTTP inside the private network
```

---

## 19. Complete Request Flow

```text
Browser opens https://example.com
    ↓
DNS → IP → TCP handshake                  1 round trip
    ↓
ClientHello → ServerHello + certificate   1 round trip (TLS 1.3)
    ↓
Certificate validated: chain, hostname, expiry, revocation
    ↓
Shared symmetric key derived
    ↓
HTTP request sent — encrypted from here on
    ↓
Load balancer decrypts, forwards plain HTTP to the application
    ↓
Response encrypted on the way back
    ↓
Session ticket stored → next connection resumes with 0 extra round trips
```

---

## 20. Key Takeaway

> [!IMPORTANT]
> TLS gives you confidentiality, integrity and proof of identity — and the identity part is why disabling certificate verification is never an acceptable workaround.

---

## 21. Common Mistakes

- **Disabling certificate verification** to get past an error
- **Letting certificates expire** — automate renewal *and* monitor it separately
- **Leaving TLS 1.0/1.1 enabled** for "old client compatibility"
- **Committing private keys** to a repository
- **Mixed content** — an HTTPS page loading HTTP resources, which browsers block
- **Terminating TLS at the load balancer and forgetting internal traffic is plaintext**
- **Confusing encryption with authorisation** — TLS proves *who*, not *what they may do*

---

## 22. Open Source Technologies

- **OpenSSL**, **BoringSSL**, **rustls** — the implementations behind almost everything
- **Let's Encrypt** with **certbot** or **Caddy** — free automated certificates
- **cert-manager** — certificate automation in Kubernetes
- **testssl.sh**, **SSL Labs** — audit your configuration
- **Linkerd**, **Istio** — automatic mTLS between services

---

## 23. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 24. Workbook Exercise

- [ ] Run `openssl s_client -connect example.com:443` and read the certificate chain it prints.
- [ ] Find every certificate your project depends on and confirm renewal is automated and monitored.
- [ ] Search your codebase for `verify=False`, `rejectUnauthorized`, or `curl -k`.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
Application (HTTP)
    ↓
TLS  (encryption + identity)
    ↓
TCP
    ↓
IP → network
```

## 2. Request Flow

```text
Input       a TCP connection and a server certificate
    ↓
Processing  negotiate, verify identity, derive a symmetric key, encrypt everything after
    ↓
Output      a confidential, tamper-evident, authenticated byte stream
```

## 3. Real-World Usage

**Let's Encrypt** issues certificates free and automatically, and single-handedly moved the web from mostly-HTTP to mostly-HTTPS. It is the clearest example in modern infrastructure of removing friction being more effective than any amount of advocacy.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | A protocol providing encryption, integrity and server identity over TCP |
| **Why does it exist?** | Because networks in between can read and rewrite plaintext traffic |
| **Where does it belong?** | Between TCP and the application protocol |
| **When should I use it?** | Everywhere — and never with verification disabled |
