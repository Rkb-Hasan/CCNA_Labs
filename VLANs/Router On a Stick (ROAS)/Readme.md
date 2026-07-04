# VLANs — Part 2: Trunking, 802.1Q & Router-on-a-Stick

**Module:** Network Fundamentals | **Simulator:** Cisco Packet Tracer  
**Source:** Jeremy's IT Lab — VLAN Part 2

> Builds on access ports and inter-VLAN basics from [Part 1](../VLAN%20Basics/README.md). Part 3 solves the same routing problem a third way, replacing the router with a Layer 3 switch — [Part 3: Multilayer Switch](../Multilayer%20Switch/README.md).

---

## Objective

Part 1 ended with three VLANs needing three separate physical links to the router — a design that doesn't scale. This lab was about collapsing that into a single trunk between two switches and one router interface carrying all three VLANs through router-on-a-stick.

---

## Theory

### Trunk Ports

An access port carries traffic for one VLAN only. A trunk port carries traffic for multiple VLANs over a single link, tagging each frame so the receiving switch knows which VLAN it belongs to. Trunks connect switches to each other, and connect a switch to a router doing inter-VLAN routing.

### Tagged vs Untagged Frames

A tagged frame carries a 4-byte VLAN tag identifying its VLAN. Frames crossing a trunk are tagged for every VLAN except the native VLAN, which crosses untagged. Access ports never see a tag at all — the switch adds it on the way out of a trunk and strips it before delivering to an access port, so the connected device has no idea VLANs exist.

### ISL vs 802.1Q

ISL (Inter-Switch Link) was Cisco's original trunking method. It wraps the entire frame in a new header and trailer, adding overhead, and only works between Cisco devices. 802.1Q is the IEEE standard that replaced it — instead of re-encapsulating the frame, it inserts a small tag directly inside the existing one. Every modern network, including this lab, uses 802.1Q.

### The 802.1Q Tag

The tag sits right after the source MAC address, before the EtherType field. It is 4 bytes: a 2-byte TPID (always `0x8100`, signalling "this frame is tagged") followed by a 2-byte TCI. The TCI splits into a 3-bit priority field, a 1-bit drop-eligible flag, and a 12-bit VLAN ID — the part that actually identifies the VLAN.

12 bits gives VLAN numbers a range of 0–4095. 0 and 4095 are reserved, leaving 1–4094 usable: 1–1005 is the normal range (1002–1005 reserved for legacy media), 1006–4094 is extended.

<img src="./Images/dot1Q header in SW1.png" width="500" alt="dot1Q_header_in_SW1">

This frame was captured on SW1 during the lab. TPID reads `0x8100`, confirming it's tagged. TCI is `0x000a` — in binary, `0000 0000 0000 1010`. Splitting off the 3-bit priority and 1-bit DEI leaves a 12-bit VID of `000000001010`, which is 10. Decoding it by hand and getting VLAN 10 back made the field layout concrete in a way the diagram alone didn't.

### Native VLAN

One VLAN per trunk is designated native — its frames cross untagged, a holdover from older equipment that didn't understand 802.1Q. Both ends of a trunk must agree on which VLAN is native; if they don't, an untagged frame arriving at one end gets placed into whatever VLAN the receiving switch thinks is native, which may not match the sender's intent. Cisco switches detect this disagreement automatically over CDP and raise a warning rather than silently misrouting traffic.

Leaving the native VLAN at the default (VLAN 1) is a known weakness, since VLAN 1 is active almost everywhere by default — exactly what a double-tagging VLAN hopping attack relies on. Pointing the native VLAN at an unused ID with no real ports removes that target.

### Router-on-a-Stick (ROAS)

ROAS routes between VLANs through a single physical router interface, split into subinterfaces — one per VLAN — each running 802.1Q encapsulation and holding the gateway IP for that VLAN. The switch port facing the router is a trunk carrying every VLAN the router needs to route between. One cable replaces the one-link-per-VLAN design from Part 1.

---

## Lab

### Addressing — carried over from Part 1

| VLAN | Name        | Subnet        | Gateway                 |
| ---- | ----------- | ------------- | ----------------------- |
| 10   | Engineering | 10.0.0.0/26   | 10.0.0.62 (R1 G0/0.10)  |
| 20   | HR          | 10.0.0.64/26  | 10.0.0.126 (R1 G0/0.20) |
| 30   | Sales       | 10.0.0.128/26 | 10.0.0.190 (R1 G0/0.30) |

### Step 1 — Access Ports

PC-facing ports on both switches were set as access ports in their correct VLAN, the same way as in [Part 1](../VLAN%20Basics/README.md).

### Step 2 — Trunk Between SW1 and SW2

```
SW1(config)# interface g0/1
SW1(config-if)# switchport mode trunk
SW1(config-if)# switchport trunk allowed vlan 10,30
SW1(config-if)# switchport trunk native vlan 27
```

<img src="./Images/Trunk and Native Vlan conf in SW1.png" width="500" alt="Trunk_and_Native_Vlan_conf_in_SW1">

Only VLANs 10 and 30 are allowed here — SW1 has no VLAN 20 devices locally, HR sits behind SW2. Native VLAN is set to 27, an unused ID with nothing assigned to it, following the security reasoning above.

### Step 3 — Router-on-a-Stick on R1

<img src="./Images/Router Sub-if config.png" width="500" alt="Router_Sub-if_config">

The physical interface carries no IP of its own — all addressing lives on the subinterfaces. `no shutdown` only runs once, on the physical interface; the subinterfaces come up automatically once it's active.

Verified with `show ip interface brief`:

<img src="./Images/Ip int brief in Router.png" width="500" alt="Ip_int_brief_in_Router">

All three subinterfaces show up/up with the correct IPs. G0/1 and G0/2 are unused this time — every VLAN now rides the single G0/0 trunk instead of needing a dedicated physical link each.

### Step 4 — Verify Trunk Status

```
show interfaces trunk
```

<img src="./Images/Sh int trunk in SW1.png" width="500" alt="Sh_int_trunk_in_SW1">

<img src="./Images/Sh int trunk in SW2.png" width="500" alt="Sh_int_trunk_in_SW2">

Both ends report native VLAN 27 and status trunking — no mismatch. SW2's link to SW1 shows encapsulation `n-802.1q`; the `n-` means it was negotiated rather than hard-set, since that side was left on `auto` instead of `on`. SW2's link to R1 carries all three VLANs, since R1 needs every one of them to route correctly.

### Step 5 — Verify VLAN Database on Both Switches

<img src="./Images/Vlan brief in SW1.png" width="500" alt="Vlan_brief_in_SW1">

<img src="./Images/VLAN brief in SW2.png" width="500" alt="VLAN_brief_in_SW2">

Both switches list VLANs 10, 20, and 30 as active. SW1 shows the auto-generated names (`VLAN0010`, `VLAN0030`) while SW2 shows the names assigned manually (`vlan10`, `vlan20`, `vlan30`) — a reminder that the VLAN database and its naming are local to each switch, not synced automatically between them.

### Step 6 — Test Connectivity

<img src="./Images/ping PC1 to PC5.png" width="500" alt="ping_PC1_to_PC5">

<img src="./Images/Ping PC4 to PC2.png" width="500" alt="Ping_PC4_to_PC2">

<img src="./Images/Ping PC5 to PC3.png" width="500" alt="Ping_PC5_to_PC3">

All three cross-VLAN pings return 4/4 with TTL 127 — one router hop, the same TTL behaviour seen since [Day 11](../../Static%20Routing/README.md). Every VLAN can now reach every other VLAN through the single trunk and R1's subinterfaces.

---

## Observations

A few things surfaced that weren't spelled out in the instructions.

VLAN 30 traffic didn't pass through SW2 until VLAN 30 was created in SW2's own VLAN database — even though it was already on both trunks' allowed list, and SW2 has no access ports in that VLAN. Allowed-vlan only filters what crosses a trunk; a switch still needs the VLAN to exist locally to handle that traffic at all, even when it's just passing through.

SW2's G0/2 (the trunk to R1) didn't show as trunking in `show interfaces trunk` until R1's physical G0/0 was brought up with `no shutdown`. Subinterfaces are entirely logical — they depend on the parent physical interface being up, and trunk negotiation happens at that physical layer first. Configuring the subinterfaces correctly meant nothing until the interface underneath them was active.

Setting SW1's native VLAN to 27 while SW2 was still on its default of VLAN 1 triggered this immediately:

<img src="./Images/Native VLAN mismatch .png" width="500" alt="Native_VLAN_mismatch_">

Seeing IOS catch the mismatch in real time, before any traffic was actually misrouted, made the security reasoning in the theory section feel a lot less abstract.

---

## Commands Reference

| Command                                | Where               | What it does                                                |
| -------------------------------------- | ------------------- | ----------------------------------------------------------- |
| `switchport mode trunk`                | Interface config    | Sets the port as a trunk instead of access                  |
| `switchport trunk allowed vlan <list>` | Interface config    | Restricts the trunk to specific VLANs                       |
| `switchport trunk native vlan <id>`    | Interface config    | Sets which VLAN crosses the trunk untagged                  |
| `show interfaces trunk`                | Privileged exec     | Shows trunk status, native VLAN, and allowed VLANs per port |
| `interface g0/0.<id>`                  | Global config       | Creates or enters a subinterface                            |
| `encapsulation dot1Q <vlan>`           | Subinterface config | Binds the subinterface to a VLAN tag                        |

Access port and VLAN creation commands carry over unchanged from [Part 1](../Part_1_VLAN_Basics/README.md).

---

## What I didn't fully understand

- [ ] DTP modes — `auto` vs `on` vs `desirable`. SW2's link to SW1 was left on `auto` and still negotiated successfully; not yet clear when `auto` alone is safe to rely on versus forcing `on` at both ends.
- [ ] Double-tagging VLAN hopping attacks in more depth — understand why an unused native VLAN closes the door on it, not yet clear on exactly how the attack frame is crafted.
- [ ] Whether VTP exists specifically to sync VLAN names and the VLAN database across switches, given the naming mismatch noticed between SW1 and SW2.

---

## Key Takeaways

ROAS collapses what used to be one physical link per VLAN into a single trunk and one router interface carrying every VLAN through subinterfaces. A VLAN has to exist in a switch's own VLAN database to be handled by that switch, regardless of whether it's allowed on a trunk or has any local access ports. Subinterfaces are logical constructs riding on a physical interface — nothing above that interface comes up until the interface itself does. Native VLAN mismatches don't fail silently; CDP flags them immediately. And decoding an actual TCI value from a captured frame by hand was a better way to learn the 802.1Q tag structure than memorising the bit layout from a diagram.
