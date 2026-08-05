# ISP, Router, Switch, Modem

> **In one line —** the four pieces of physical plumbing between your laptop and the internet, each doing one distinct job that people constantly confuse.

| | |
|---|---|
| **Category** | Network Hardware |
| **Architectural Layer** | Link / Network |
| **Related notes** | [Internet Fundamentals](Internet%20Fundamentals.md) · [IP Addresses](IP%20Addresses.md) · [TCP IP](TCP%20IP.md) · [Load Balancing](../14%20-%20Scalability%20and%20Reliability/Load%20Balancing.md) |

---

## 1. The four roles at a glance

| Device | Job | Works with |
|---|---|---|
| **Modem** | Translates between your ISP's physical medium and Ethernet | Signals |
| **Router** | Decides where a packet goes **between** networks | IP addresses |
| **Switch** | Delivers frames **within** one network | MAC addresses |
| **ISP** | Provides your connection to the rest of the internet | Everything |

> [!IMPORTANT]
> **Switch = inside one network. Router = between networks.** That one distinction explains almost everything else on this page.

---

## 2. The path from your laptop to a server

```text
Your laptop
    ↓  Wi-Fi / Ethernet
SWITCH        (delivers within your local network, by MAC address)
    ↓
ROUTER        (decides: local, or out to the internet? Performs NAT)
    ↓
MODEM         (converts Ethernet to the ISP's medium — fibre, cable, DSL)
    ↓
ISP network
    ↓
Backbone routers, undersea cables, peering exchanges
    ↓
Destination data centre → its router → its switch → the server
```

In a home, the "router" you own is usually all three devices in one box: modem, router, switch and Wi-Fi access point.

---

## 3. Modem

*What is it?*

A **mod**ulator–**dem**odulator converts digital data into whatever signal the physical line carries — light for fibre, radio frequencies for cable, electrical tones for DSL — and back again.

It does not understand IP addresses. It only translates between media.

---

## 4. Router

*What is it?*

A router connects **different networks** and decides where each packet goes next. It reads the destination [IP address](IP%20Addresses.md), consults its routing table, and forwards the packet one hop closer.

```text
Packet arrives
    ↓
Is the destination on a network I am directly connected to?
    ↓ YES → deliver locally
    ↓ NO  → forward to the next router toward it
```

Your home router also performs **NAT** (Network Address Translation): every device inside shares one public IP address, and the router remembers which internal device each connection belongs to.

> [!TIP]
> NAT is the reason your laptop has a private address like `192.168.1.7` and why incoming connections from the internet do not reach it unless you explicitly forward a port.

---

## 5. Switch

*What is it?*

A switch connects devices **inside one network** and forwards frames by **MAC address** — a hardware identifier burned into each network card. It learns which device is on which physical port and sends each frame only where it needs to go.

```text
HUB (obsolete)      copies every frame to every port — wasteful and insecure
    ↓
SWITCH              learns MAC addresses, sends each frame only to its destination port
```

---

## 6. ISP

*What is it?*

An Internet Service Provider owns network infrastructure and sells you access to it. ISPs interconnect with each other at **peering points**, and those interconnections are what make "the internet" a single reachable network rather than a set of islands.

```text
You → local ISP → regional network → backbone (Tier 1) → destination ISP → server
```

> [!IMPORTANT]
> No ISP owns the internet. Each owns a piece and agrees to carry traffic for the others. The internet is a commercial and technical agreement, not a thing anyone built.

---

## 7. Why this matters to a backend developer

Most of this is invisible day to day, until it is not:

- **NAT and firewalls** explain why a service reachable from your laptop is unreachable from a colleague's, and why webhooks need a public endpoint or a tunnel.
- **MTU and packet fragmentation** cause bizarre failures where small requests work and large ones hang — common with VPNs and some container networks.
- **Cloud VPC design** ([VPC](../12%20-%20Cloud%20Architecture/VPC.md)) is exactly this model in software: subnets, route tables, gateways.
- **"It works locally"** is very often a routing or NAT difference, not a code difference.

---

## 8. Real World Example

- **A Kubernetes cluster** implements all of this in software: virtual switches for pod-to-pod traffic, virtual routers between nodes, and NAT for outbound internet access.
- **AWS VPC** gives you subnets (switching domains), route tables (routing), internet gateways and NAT gateways — the same four ideas, rented.
- **A home lab** exposing a service to the internet requires port forwarding on the router, which is NAT being told what to do with unsolicited inbound connections.

---

## 9. Mental Model

> [!NOTE]
> **Think of a city's postal system.**
>
> The **switch** is the mail room inside one building — it knows every desk. The **router** is the local post office — it does not know your desk, only which office to send the parcel to next. The **modem** is the loading dock, converting between the van and the building. The **ISP** is the courier company that owns the vans and agrees with other companies to carry each other's parcels.

---

## 10. Mini Architecture Diagram

```text
┌──────── your local network ────────┐
│  Laptop  Phone  Printer            │
│     └───────┬───────┘              │
│          SWITCH   (MAC addresses)  │
│             ↓                      │
│          ROUTER   (IP + NAT)       │
└─────────────┬──────────────────────┘
              ↓
           MODEM   (signal conversion)
              ↓
            ISP
              ↓
         The internet
```

---

## 11. Key Takeaway

> [!IMPORTANT]
> A switch moves data inside one network by MAC address; a router moves it between networks by IP address; a modem converts signals; an ISP connects you to everyone else.

---

## 12. Common Mistakes

- **Confusing switches and routers** — the layer they work at is the whole difference
- **Assuming a private IP is reachable from the internet** — NAT means it is not
- **Ignoring MTU issues** when debugging VPN or container networking problems
- **Believing "the internet is down"** when it is usually DNS or one ISP's route
- **Treating cloud networking as different** — VPCs are these same concepts in software

---

## 13. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 14. Workbook Exercise

- [ ] Find your device's private IP and your public IP, and explain why they differ.
- [ ] Run `traceroute` to a foreign server and identify roughly where your ISP hands off to a backbone network.
- [ ] Draw your own home or office network, labelling which device performs which of the four roles.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
Device → Switch → Router → Modem → ISP → Internet → destination network
```

## 2. Request Flow

```text
Input       a packet with a destination IP
    ↓
Processing  switched locally by MAC, routed globally by IP, converted for the physical line
    ↓
Output      the packet delivered one hop closer to its destination
```

## 3. Real-World Usage

**AWS VPC** reproduces this entire model in software. Subnets, route tables, internet gateways and NAT gateways are the switch, router, ISP link and NAT translation of a physical network, rented by the hour.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | The four devices connecting a machine to the wider internet |
| **Why does it exist?** | Because local delivery, inter-network routing and signal conversion are different jobs |
| **Where does it belong?** | Between every client and every server, physically |
| **When should I use it?** | Understand it when debugging connectivity, NAT and cloud networking |
