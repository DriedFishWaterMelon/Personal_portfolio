# Lab 2 — Secure & Scalable VLAN Design (Router-on-a-Stick)

**Course:** Network Architecture  
**Tool:** Cisco Packet Tracer  
**File:** `lab2.pkt`

---

## Team

| Name | Student ID | Part |
|---|---|---|
| เพชรภิญโญ ธนศิรินรากร | 673380073-7 | Parts 1–4 Network Setup |
| ก้องภพ โชควิริยะ | 673380030-5 | Part 5 — Scenario A |
| ณพวิทย์ วงษ์ประเสริฐ | 673380266-6 | Part 5 — Scenario C |
| รัฐภูมิ เกิดพระจีน | 673380057-5 | Part 5 — Scenario B |
| อชิรวัช บึงไสย์ | 673380298-3 | Worksheets 1 & 2 + Assessment Questions |

---

## Overview

This lab implements a **Router-on-a-Stick** inter-VLAN routing architecture — a single physical router interface carrying multiple VLANs via 802.1Q subinterfaces. The design segments traffic into three logical networks, isolates management, and locks down unused switch ports to a blackhole VLAN for security.

---

## Network Topology

```
         [ R1 – Router ]
           G0/0 (802.1Q Trunk)
               |
         [ S1 – Switch ]
        /       |       \
     Fa0/1    Fa0/2    Fa0/3
       |        |        |
    [PC-A]   [PC-B]   [PC-C]
    VLAN10   VLAN20   VLAN99
```

---

## Addressing Table

| Device | Interface | VLAN | IP Address | Subnet Mask | Default Gateway |
|---|---|---|---|---|---|
| R1 | G0/0.10 | 10 | 192.168.10.1 | 255.255.255.192 | N/A |
| R1 | G0/0.20 | 20 | 192.168.10.65 | 255.255.255.192 | N/A |
| R1 | G0/0.99 | 99 | 192.168.10.129 | 255.255.255.192 | N/A |
| S1 | VLAN 99 | 99 | 192.168.10.131 | 255.255.255.192 | 192.168.10.129 |
| PC-A | NIC | 10 | 192.168.10.10 | 255.255.255.192 | 192.168.10.1 |
| PC-B | NIC | 20 | 192.168.10.70 | 255.255.255.192 | 192.168.10.65 |
| PC-C | NIC | 99 | 192.168.10.130 | 255.255.255.192 | 192.168.10.129 |

**Major Network:** `192.168.10.0/24` — Subnetted into /26 blocks (64 hosts each)

| VLAN | Name | Subnet |
|---|---|---|
| 10 | USERS | 192.168.10.0/26 |
| 20 | SERVERS | 192.168.10.64/26 |
| 99 | MANAGEMENT | 192.168.10.128/26 |
| 999 | BLACKHOLE | (unused ports) |

---

## Part 1 — VLAN Configuration on Switch

Created VLANs and assigned ports:

```
S1(config)# vlan 10
S1(config-vlan)# name USERS
S1(config)# vlan 20
S1(config-vlan)# name SERVERS
S1(config)# vlan 99
S1(config-vlan)# name MANAGEMENT
S1(config)# vlan 999
S1(config-vlan)# name BLACKHOLE
```

Port assignments: Fa0/1 → VLAN 10, Fa0/2 → VLAN 20, Fa0/3 → VLAN 99

`show vlan brief` confirmed all four VLANs active with correct port bindings.

---

## Part 2 — Router-on-a-Stick (Inter-VLAN Routing)

Configured subinterfaces on R1's single G0/0 interface:

```
R1(config)# interface G0/0.10
R1(config-subif)# encapsulation dot1Q 10
R1(config-subif)# ip address 192.168.10.1 255.255.255.192

R1(config)# interface G0/0.20
R1(config-subif)# encapsulation dot1Q 20
R1(config-subif)# ip address 192.168.10.65 255.255.255.192

R1(config)# interface G0/0.99
R1(config-subif)# encapsulation dot1Q 99
R1(config-subif)# ip address 192.168.10.129 255.255.255.192
```

`show ip interface brief` confirmed all three subinterfaces UP/UP.

---

## Part 3 — Connectivity Verification (/26 Subnetting)

Cross-VLAN pings confirmed inter-VLAN routing works correctly via R1.

| From | To | Result | TTL |
|---|---|---|---|
| PC-A → R1 G0/0.10 | 192.168.10.1 | ✅ 4/4 | 255 |
| PC-A → PC-B | 192.168.10.70 | ✅ 4/4 | 127 |
| PC-A → S1 Management | 192.168.10.131 | ✅ 4/4 | 254 |

TTL of 127 on PC-A → PC-B indicates the packet passed through R1 (one router hop).

---

## Part 4 — Security: Management Isolation & Port Lockdown

Unused switch ports (Fa0/4 through Fa0/24) were shut down and assigned to VLAN 999 (BLACKHOLE):

```
S1(config)# interface range fa0/4-24
S1(config-if-range)# switchport mode access
S1(config-if-range)# switchport access vlan 999
S1(config-if-range)# shutdown
```

The trunk port (Gig0/1) was restricted to only allow VLANs 10, 20, and 99:
```
S1(config-if)# switchport trunk allowed vlan 10,20,99
```

---

## Part 5 — Troubleshooting Scenarios

### Scenario A — Wrong VLAN Assignment
- **Fault:** PC-B's switch port moved from VLAN 20 to VLAN 10
- **Symptom:** `ping 192.168.10.65` (VLAN 20 gateway) → 100% loss from PC-B
- **Root Cause:** PC-B is now in VLAN 10's broadcast domain — it cannot reach its own gateway in VLAN 20
- **Fix:** `switchport access vlan 20` on Fa0/2

### Scenario B — Missing VLAN on Trunk
- **Fault:** VLAN 99 removed from trunk with `switchport trunk allowed vlan 10,20`
- **Symptom:** `ping 192.168.10.131` (S1 management IP) → 100% loss
- **Root Cause:** VLAN 99 traffic is dropped at the trunk — it cannot reach R1 for routing
- **Fix:** `switchport trunk allowed vlan add 99`

### Scenario C — Wrong Default Gateway
- **Fault:** PC-A gateway changed to `192.168.10.65` (VLAN 20's gateway, wrong for VLAN 10)
- **Symptom:** Cannot ping across subnets; 25% packet loss observed
- **Root Cause:** PC-A cannot ARP for the correct MAC address of its gateway, so packets destined for other networks are dropped
- **Fix:** Reset gateway to `192.168.10.1`

---

## Lab Completion Checklist

| Checkpoint | Command | Expected | Result |
|---|---|---|---|
| VLANs exist | `show vlan brief` | VLANs 10, 20, 99 | ✅ PASS |
| Trunk active | `show interfaces trunk` | VLANs 10,20,99 allowed | ✅ PASS |
| Router subinterfaces | `show ip int brief` | All subifs up/up | ✅ PASS |
| Inter-VLAN ping | `ping` across VLANs | Success | ✅ PASS |

---

## Assessment Answers

1. **Which device performs inter-VLAN routing?** R1 — configured as a Router-on-a-Stick with 802.1Q subinterfaces (G0/0.10, G0/0.20, G0/0.99).

2. **Why can PCs in different VLANs communicate?** Because R1 acts as the Layer 3 default gateway for each VLAN. It receives frames from one VLAN subinterface and routes them out through another.

3. **What breaks if the trunk is removed?** All inter-VLAN communication fails immediately. VLANs are isolated at Layer 2 — without a trunk, their traffic cannot reach R1 for routing.

4. **Why is VLAN 1 avoided?** VLAN 1 is the factory default and a well-known target for VLAN hopping attacks. Using a custom management VLAN (99) reduces exposure.

5. **Security risk of leaving unused ports active?** An attacker could plug in a device and gain access to the network. Assigning unused ports to VLAN 999 (no gateway, no route out) contains the blast radius even if someone connects.

**Why VLAN 99 for management?** It separates management traffic (SSH, SNMP) from user data, making it harder to intercept or spoof control-plane communications.

**Why does each VLAN need a different subnet?** VLANs segment Layer 2 broadcast domains. Without matching Layer 3 subnets, hosts in different VLANs would have no routable path to each other — the router needs a distinct network address per VLAN to build its routing table.

**What does 802.1Q tagging do?** Inserts a 4-byte VLAN ID tag into the Ethernet frame header so a single trunk link can carry traffic from multiple VLANs without mixing them.

---

## Files
- `__Lab_2__Secure___Scalable_VLAN_Design__Router-on-a-Stick_.pdf` — Full lab report
- `lab2.pkt` — Packet Tracer simulation file
