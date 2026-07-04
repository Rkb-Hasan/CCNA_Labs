# VLANs — Index

**Module:** Network Fundamentals  
**Source:** Jeremy's IT Lab — VLAN Parts 1, 2 & 3

---

## What This Section Covers

Without VLANs, every device on a switch shares the same broadcast domain regardless of which subnet it belongs to. VLANs fix that by creating logical separation at Layer 2 — isolating broadcast traffic, improving security, and making networks easier to manage without adding physical hardware.

But separation creates a new problem: devices in different VLANs can no longer reach each other directly. The three parts in this section each solve that problem a different way, starting simple and ending with the approach used in most modern networks.

---

## The Three Approaches

| Part                                                                         | Method                                    | Key Idea                                         |
| ---------------------------------------------------------------------------- | ----------------------------------------- | ------------------------------------------------ |
| [Part 1 — VLAN Basics & Separate Links](./VLAN%20Basics/README.md)           | One physical link per VLAN to a router    | Simple to understand, does not scale             |
| [Part 2 — Router-on-a-Stick](<./Router%20On%20a%20Stick%20(ROAS)/Readme.md>) | One trunk link, router subinterfaces      | Scales better, still depends on external router  |
| [Part 3 — Multilayer Switch & SVI](./Multilayer%20Switch/README.md)          | Layer 3 switch routes internally via SVIs | No external router needed for inter-VLAN traffic |

Read them in order — each part assumes the previous one.

---

## How the Three Parts Connect

Part 1 sets the foundation. VLANs are configured, ports are assigned, and a router handles inter-VLAN routing through one dedicated link per VLAN. It works, but three VLANs already consumed three router interfaces and three switch uplinks.

Part 2 solves the scaling problem. The three links collapse into one trunk, and the router uses subinterfaces — one per VLAN — to keep routing between them. The logic is the same as Part 1 but the physical design is much leaner.

Part 3 removes the router from inter-VLAN routing entirely. A multilayer switch takes over that job internally through SVIs, each one acting as the gateway for its VLAN. The router stays connected but only handles traffic leaving the local network — internet-bound requests and nothing else.

---

## Lab Topology

All three labs use the same three VLANs and the same address space. The topology evolves across the parts rather than starting fresh each time.

| VLAN | Name        | Subnet        |
| ---- | ----------- | ------------- |
| 10   | Engineering | 10.0.0.0/26   |
| 20   | HR          | 10.0.0.64/26  |
| 30   | Sales       | 10.0.0.128/26 |
