# Day 11 — Troubleshooting Static Routes

**Date:** 2026-05-24 | **Module:** Network Fundamentals | **Simulator:** Cisco Packet Tracer  
**Source:** Jeremy's IT Lab

> This lab uses the same topology and addressing from [README.md](./README.md), where those routes were first configured correctly. Coming back to break and fix them was a good test of whether the concepts actually stuck.

---

## Objective

PC1 and PC2 could not ping each other. There was one misconfiguration on each router — R1, R2, and R3. The task was to find each fault using `show` commands, understand why it broke connectivity, fix it, and verify the result.

---

## Starting Point

Before touching anything, `show ip route` was run on all three routers to get a picture of what was in each routing table. The faults were not immediately obvious just from reading the output — the routes existed, they just pointed to the wrong places.

---

## Fault 1 — R1: Wrong Next-Hop IP

### What was observed

R1's routing table showed a static route for 192.168.3.0/24, which is correct. But looking closely at the next-hop IP, it pointed to 192.168.12.3 instead of 192.168.12.2.

<img src="./Troubleshoot_Images/R1 Faulty Hop IP.png" width="500" alt="R1_Faulty_Hop_IP">

192.168.12.3 does not exist on the R1-R2 link. R2's G0/0 is 192.168.12.2. So every packet R1 tried to forward toward PC2 was being sent to an address that has no device behind it — which means it was just dropped.

### The fix

The wrong route had to be removed first, then the correct one added:

```
R1(config)# no ip route 192.168.3.0 255.255.255.0 192.168.12.3
R1(config)# ip route 192.168.3.0 255.255.255.0 192.168.12.2
```

### Verified

<img src="./Troubleshoot_Images/R1 IP Routes.png" width="500" alt="R1_IP_Routes">

The S entry now shows via 192.168.12.2 — R2's actual G0/0 address. R1 is fixed.

---

## Fault 2 — R2: Wrong Exit Interface

### What was observed

R2's static route for 192.168.3.0/24 was configured using an exit interface rather than a next-hop IP. The interface it pointed to was G0/0, which connects back toward R1 — the wrong direction entirely. Traffic heading to PC2's network was being sent back toward PC1's side.

<img src="./Troubleshoot_Images/R2 Faulty exit interface.png" width="500" alt="R2_Faulty_exit_interface">

G0/1 is the interface that connects R2 to R3. That is where this route should have been pointing.

### The fix attempt — and something unexpected

The corrected route was added:

```
R2(config)# ip route 192.168.3.0 255.255.255.0 g0/1
```

But the old route via G0/0 did not disappear. Looking at the routing table after, both routes were present at the same time:

<img src="./Troubleshoot_Images/Load balancing of R2.png" width="500" alt="Load_balancing_of_R2">

This was my the first time seeing **load balancing** in the routing table. When two static routes point to the same destination network and have equal Administrative Distance and metric — in this case both are [1/0] — Cisco IOS keeps both and distributes traffic across them. Neither one overrides the other.

The old incorrect route had to be removed manually:

```
R2(config)# no ip route 192.168.3.0 255.255.255.0 g0/0
```

### A warning that appeared

When the corrected route was added using the exit interface, Cisco IOS printed this message:

<img src="./Troubleshoot_Images/Did not understand fully.png" width="500" alt="Did_not_understand_fully">

Later Found out, this message comes up because on an Ethernet link — which is a multi-access network, meaning multiple devices can exist on it — the router needs to know not just which interface to use, but also which specific device on that interface to send to. That requires a next-hop IP so the router can ARP for the destination MAC address.

### Verified

<img src="./Troubleshoot_Images/R2 IP Routes.png" width="500" alt="R2_IP_Routes">

After removing the G0/0 route, R2 now has one clean S entry for 192.168.3.0/24 via G0/1.

---

## Fault 3 — R3: Wrong IP on G0/0

### What was observed

R3's routing table had an S entry pointing to 192.168.1.0/24 via 192.168.13.2, which looked correct. But the static route in R3 depends on its G0/0 interface being on the 192.168.13.0/24 network to actually reach R2.

Running `show ip interface brief` revealed the problem:

<img src="./Troubleshoot_Images/R3 Faulty IP Config interface.png" width="500" alt="R3_Faulty_IP_Config_interface">

G0/0 had been configured with 192.168.23.3 instead of 192.168.13.3. So R3 and R2 were on completely different networks — there was no Layer 3 path between them at all. Any packet R3 tried to forward via 192.168.13.2 would never reach R2 because R3 itself was not on the 192.168.13.0/24 network.

### The fix

The correct IP was configured on G0/0:

```
R3(config)# interface g0/0
R3(config-if)# ip address 192.168.13.3 255.255.255.0
```

### Something worth noting

Unlike the R2 route situation where adding a new route did not remove the old one, the new IP address here replaced the old one immediately. No manual removal was needed. Interface IP configuration overwrites — routing table entries do not.

### Verified

<img src="./Troubleshoot_Images/R3 IP Routes.png" width="500" alt="R3_IP_Routes">

R3's C and L entries now correctly show 192.168.13.0/24 on G0/0, and the static route to 192.168.1.0/24 via 192.168.13.2 is reachable. R3 is fixed.

---

## Final Verification

With all three faults corrected, the ping from PC1 to PC2 was tested:

<img src="./Troubleshoot_Images/Ping reply from PC1 to PC2.png" width="500" alt="Ping_reply_from_PC1_to_PC2">

4 out of 4 replies, 0% packet loss, TTL 125. Same result as the working lab in [README.md](./README.md).

---

## What I didn't understand

- [ ] Load balancing in more depth — Cisco IOS was distributing packets across two paths. Does it alternate per packet or per flow?
- [ ] The Proxy ARP warning — the surface-level explanation makes sense now (Ethernet needs a next-hop IP to ARP correctly), but the actual mechanics of how Proxy ARP resolves this behind the scenes are still not fully clear.

---

## Key Takeaways

- `show ip route` is the first command to run when troubleshooting a routing problem. The fault is usually visible right there — a wrong next-hop, a wrong interface, or a missing route entirely.
- Adding a corrected route does not remove the old one. Routes with the same destination but different next-hops or interfaces are treated as separate entries and Cisco IOS will load-balance across them. The wrong route must be explicitly removed with `no ip route`.
- Interface IP configuration behaves differently — assigning a new IP on an interface replaces the old one without any manual removal.
- On Ethernet links, static routes should always use a next-hop IP rather than just an exit interface. The warning Cisco IOS printed when the interface-only route was added was a signal that something was not quite right, even before understanding exactly why.
