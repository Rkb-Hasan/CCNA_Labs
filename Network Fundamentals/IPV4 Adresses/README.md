# IPv4 Addressing & Router Interface Configuration

**Date:** 2026-05-12 | **Module:** Network Fundamentals | **Simulator:** Cisco Packet Tracer  
**Source:** Jeremy's IT Lab

---

## Objective

Understand the structure of IPv4 addresses — how they are written, classed, and divided into network vs host portions — then apply that knowledge by manually configuring IP addresses on a Cisco router's interfaces, verifying connectivity, and saving the configuration.

---

## Theory

### What is an IPv4 Address?

An IPv4 address is a **32-bit number** used to identify a device on a network. It is written in **dotted decimal notation** — four groups of 8 bits (called **octets**), each converted to decimal and separated by a dot.

```
Binary:   11000000 . 10101000 . 00000001 . 00000001
Decimal:    192    .   168    .    1     .    1
```

- Total address space: **2³² = 4,294,967,296** unique addresses
- Each octet ranges from **0 to 255**

---

### Network Portion vs Host Portion

Every IPv4 address is split into two parts:

| Part                | Purpose                                            |
| ------------------- | -------------------------------------------------- |
| **Network portion** | Identifies which network the device belongs to     |
| **Host portion**    | Identifies the specific device within that network |

The split is defined by the **subnet mask** (or prefix length).

- `192.168.1.10 / 255.255.255.0` → first 3 octets = network, last octet = host
- The same thing written as a **prefix**: `192.168.1.10/24` (24 bits = network)

---

### IPv4 Address Classes

Historically, addresses were divided into classes based on the **leading bit pattern** of the first octet. This determines the default network and host portions.

| Class | Leading Bits | First Octet Range | Default Prefix | Max Networks | Max Hosts per Network | Purpose                 |
| ----- | ------------ | ----------------- | -------------- | ------------ | --------------------- | ----------------------- |
| **A** | `0xxxxxxx`   | 1 – 126           | /8             | 126          | 16,777,214            | Large organisations     |
| **B** | `10xxxxxx`   | 128 – 191         | /16            | 16,384       | 65,534                | Medium organisations    |
| **C** | `110xxxxx`   | 192 – 223         | /24            | 2,097,152    | 254                   | Small networks          |
| **D** | `1110xxxx`   | 224 – 239         | N/A            | N/A          | N/A                   | Multicast               |
| **E** | `1111xxxx`   | 240 – 255         | N/A            | N/A          | N/A                   | Experimental / reserved |

> **Max hosts formula:** `2ⁿ - 2` where `n` = number of host bits.  
> Subtract 2 because the **network address** (all host bits = 0) and **broadcast address** (all host bits = 1) are reserved and cannot be assigned to a device.

#### Why 127 is missing from Class A

`127.x.x.x` is reserved for **loopback** — traffic sent here never leaves the device.  
`127.0.0.1` is the most well-known loopback address. Used to test the TCP/IP stack on the local machine.

---

### Special Address Ranges to Know

| Range                         | Type              | Notes                                              |
| ----------------------------- | ----------------- | -------------------------------------------------- |
| `0.0.0.0/8`                   | "This network"    | Used before an IP is assigned (e.g. DHCP DISCOVER) |
| `127.0.0.0/8`                 | Loopback          | Never routed; stays on local device                |
| `224.0.0.0 – 239.255.255.255` | Multicast         | Class D; sent to a group of devices                |
| `240.0.0.0 – 255.255.255.255` | Experimental      | Class E; not used in production                    |
| `255.255.255.255`             | Limited broadcast | Sent to all devices on the local segment           |

---

### Layer Interactions — a quick reminder

| Term                           | What it means                                                                                                                                        |
| ------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Adjacent layer interaction** | One layer on a device hands data to the layer directly above or below it (e.g. Network layer passes a packet down to Network Access to be framed)    |
| **Same layer interaction**     | The same layer on two different devices communicates using a shared protocol (e.g. IP on Router A sets the destination IP that IP on Router B reads) |

---

## 🖧 Lab — Configuring R1's Interfaces

### Topology

![Network Topology](topology.png)

| Interface | Connected to | Network         | IP Address Assigned |
| --------- | ------------ | --------------- | ------------------- |
| G0/0      | SW1 → PC1    | 15.0.0.0/8      | 15.255.255.254      |
| G0/1      | SW2 → PC2    | 182.98.0.0/16   | 182.98.255.254      |
| G0/2      | SW3 → PC3    | 201.191.20.0/24 | 201.191.20.254      |

> A best practice is to assign the **highest usable host address** (x.x.x.254) to the router — it's not required, but it's consistent and easy to remember.

---

### Step 1 — Check interfaces before configuration

Before touching anything, run a quick look at what the router already has:

```
R1# show ip interface brief
```

At this point all interfaces show `administratively down` / `down` — the default state for router interfaces (unlike switches, **router interfaces are shutdown by default**).

---

### Step 2 — Configure IP addresses and enable interfaces

Enter interface config mode, assign the IP, add a description, and bring the interface up with `no shutdown`.

```
R1(config)# hostname R1

R1(config)# interface gigabitEthernet 0/0
R1(config-if)# ip address 15.255.255.254 255.0.0.0
R1(config-if)# description ##to SW1##
R1(config-if)# no shutdown

R1(config)# interface gigabitEthernet 0/1
R1(config-if)# ip address 182.98.255.254 255.255.0.0
R1(config-if)# description #to SW2
R1(config-if)# no shutdown

R1(config)# interface gigabitEthernet 0/2
R1(config-if)# ip address 201.191.20.254 255.255.255.0
R1(config-if)# description ##to SW3##
R1(config-if)# no shutdown
```

![Setting IP and description on G0/1](setting_up_IP___description.png)

---

### Step 3 — Verify

#### `show ip interface brief` — quick summary of all interfaces

```
R1# show ip interface brief
```

![show ip interface brief output](show_ip_interfaces.png)

What each column means:

| Column         | Meaning                                                                                                                                     |
| -------------- | ------------------------------------------------------------------------------------------------------------------------------------------- |
| **Interface**  | Physical interface name                                                                                                                     |
| **IP-Address** | Configured IP (or `unassigned`)                                                                                                             |
| **OK?**        | `YES` = IP address is valid                                                                                                                 |
| **Method**     | How the IP was set (`manual` = we typed it; `unset` = nothing configured)                                                                   |
| **Status**     | **Layer 1** — physical link status. `up` = cable connected and active. `administratively down` = manually shut down with `shutdown` command |
| **Protocol**   | **Layer 2** — data link status. `up` = line protocol is active (can send/receive frames). Only goes `up` if Layer 1 is also `up`            |

> `Vlan1` shows `administratively down / down` — normal, we didn't configure it and it's irrelevant for this router lab.

---

#### `show interfaces` — detailed stats for all interfaces

```
R1# show interfaces
```

![show interfaces output](show_interfaces.png)

Key fields to know from this output:

| Field                                                        | What it tells you                                         |
| ------------------------------------------------------------ | --------------------------------------------------------- |
| `GigabitEthernet0/0 is up, line protocol is up`              | L1 and L2 both up                                         |
| `Hardware is CN Gigabit Ethernet, address is 0060.3e72.3101` | MAC address of the interface                              |
| `Internet address is 15.255.255.254/8`                       | Configured IP with prefix                                 |
| `MTU 1500 bytes`                                             | Maximum Transmission Unit — largest frame payload allowed |
| `BW 1000000 Kbit`                                            | Interface bandwidth (1 Gbps here)                         |
| `Full-duplex, 100Mb/s`                                       | Duplex and actual negotiated speed                        |
| `Encapsulation ARPA`                                         | Ethernet II framing (standard for LAN)                    |
| `reliability 255/255`                                        | 255/255 = 100% reliable (no errors)                       |
| `input errors, CRC`                                          | Useful for troubleshooting bad cables/hardware            |

---

#### `show interfaces g0/0` — same output but filtered to one interface

```
R1# sh int g0/0
```

![show interface g0/0 output](show_interface_gigabit.png)

Use this when you want detail on a specific interface without scrolling through all of them.

---

### Step 4 — Save the configuration

```
R1# copy running-config startup-config
```

or the shortcut:

```
R1# write memory
```

> **Running config** = what's active right now (lives in RAM, lost on reboot).  
> **Startup config** = saved to NVRAM, reloads on boot.  
> Always save after making changes you want to keep.

---

### Step 5 — Configure PC IP addresses

Set each PC's IP, subnet mask, and default gateway in Packet Tracer:

| Device | IP Address   | Subnet Mask   | Default Gateway |
| ------ | ------------ | ------------- | --------------- |
| PC1    | 15.0.0.1     | 255.0.0.0     | 15.255.255.254  |
| PC2    | 182.98.0.1   | 255.255.0.0   | 182.98.255.254  |
| PC3    | 201.191.20.1 | 255.255.255.0 | 201.191.20.254  |

> The **default gateway** is always the router interface IP on the same network as that PC.

---

### Step 6 — Test connectivity

```
PC1> ping 182.98.0.1   (PC2)
PC1> ping 201.191.20.1 (PC3)
```

Expected result: `!!!!!` (5 replies, 0% loss)

---

### What happened under the hood during the ping

When PC1 pings PC3 for the first time:

1. **ARP** — PC1 doesn't know the router's MAC, so it sends an ARP broadcast on the 15.0.0.0/8 segment. R1 replies with its G0/0 MAC.
2. **Switch MAC table** — SW1 learns PC1's MAC on the port it arrived from, then forwards R1's ARP reply to PC1.
3. **ICMP Echo** — PC1 wraps the ping in an IP packet (src: 15.0.0.1, dst: 201.191.20.1), frames it with R1's MAC as destination, and sends it.
4. **Router** — R1 strips the Ethernet frame (Layer 2 decapsulation), reads the IP destination, looks up its routing table, then **re-frames** the packet with a new Ethernet header for the G0/2 segment and forwards it to SW3 → PC3.
5. **MAC changes, IP stays the same** — the IP packet header is identical end-to-end; only the Ethernet frame is rebuilt at each hop.

---

## Commands Reference

| Command                              | Where            | What it does                                                        |
| ------------------------------------ | ---------------- | ------------------------------------------------------------------- |
| `hostname R1`                        | Global config    | Sets the device name shown in the prompt                            |
| `interface g0/0`                     | Global config    | Enters interface configuration mode                                 |
| `ip address <ip> <mask>`             | Interface config | Assigns an IP address and subnet mask                               |
| `description <text>`                 | Interface config | Adds a human-readable label to the interface                        |
| `no shutdown`                        | Interface config | Brings the interface up (removes admin shutdown)                    |
| `show ip interface brief`            | Privileged exec  | Summary table: IP, status (L1), protocol (L2) for all interfaces    |
| `show interfaces`                    | Privileged exec  | Detailed stats for all interfaces (MAC, MTU, errors, duplex, speed) |
| `show interfaces g0/0`               | Privileged exec  | Same detail but for one specific interface only                     |
| `show running-config`                | Privileged exec  | Shows the current active configuration in RAM                       |
| `copy running-config startup-config` | Privileged exec  | Saves running config to NVRAM (persists reboots)                    |
| `write memory`                       | Privileged exec  | Shortcut for the above                                              |

---

## What I didn't fully understand

- [ ] Why `Vlan1` shows up on a router's `show ip int brief` — need to revisit
- [ ] Exactly what ARPA encapsulation means vs other types
- [ ] The full ICMP/ARP exchange in simulation mode — only partially observed

---

## Key Takeaways

- **Router interfaces are shutdown by default** — always add `no shutdown` after configuring an IP or the interface stays down.
- `show ip int brief` — go-to command for a fast health check. **Status = Layer 1, Protocol = Layer 2.**
- **MAC addresses change at every hop; IP addresses stay the same end-to-end.** The router strips the incoming frame and builds a brand new one for the outgoing segment.
- Always save with `copy run start` or `write mem` — RAM doesn't survive a reboot.
