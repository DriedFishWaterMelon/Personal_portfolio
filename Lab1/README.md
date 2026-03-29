# Lab 1 — Protocol Stack & Encapsulation Analysis

**Course:** Network Architecture  
**Tool:** Cisco Packet Tracer  
**File:** `ITN_Module3_ProtocolLab.pkt`

---

## Team

| Name | Student ID | Part |
|---|---|---|
| เพชรภิญโญ ธนศิรินรากร | 673380073-7 | Part 1 — Device Configuration + Bonus |
| ก้องภพ โชควิริยะ | 673380030-5 | Part 2 — Connectivity Verification |
| ณพวิทย์ วงษ์ประเสริฐ | 673380266-6 | Part 3 — Encapsulation Analysis |
| รัฐภูมิ เกิดพระจีน | 673380057-5 | Part 4 — Troubleshooting Challenge |
| อชิรวัช บึงไสย์ | 673380298-3 | Part 5 — Advanced Observation |

---

## Overview

A foundational networking lab that builds a single LAN with a router, switch, and two PCs — then uses Cisco Packet Tracer's simulation mode to observe the full OSI encapsulation stack in action. The lab progresses from device configuration through protocol analysis, deliberate fault injection, and recovery.

---

## Network Topology

```
        [Router0 – ISR2911]
               |
          [Switch0 – 2960]
         /       |       \
   [Server0]  [PC1]    [PC2]
```

---

## Addressing Table

| Device | Interface | IP Address | Subnet Mask | Default Gateway | MAC |
|---|---|---|---|---|---|
| R1 | G0/0/0 | 192.168.1.1 | 255.255.255.0 | N/A | 00D0.BC6A.3501 |
| S1 | VLAN 1 | 192.168.1.2 | 255.255.255.0 | 192.168.1.1 | 00D0.BA1B.D8EC |
| PC1 | NIC | 192.168.1.10 | 255.255.255.0 | 192.168.1.1 | 0060.2F28.0395 |
| PC2 | NIC | 192.168.1.11 | 255.255.255.0 | 192.168.1.1 | 000D.BDC4.3914 |
| Server0 *(Bonus)* | NIC | 192.168.1.100 | 255.255.255.0 | 192.168.1.1 | 00D0.FFD1.C4D0 |

**Network:** `192.168.1.0/24` — Usable: `.1–.254` — Broadcast: `.255`

---

## Part 1 — Device Configuration

Configured static IP addresses on R1, S1, PC1, and PC2. The bonus task added Server0 at `192.168.1.100` for HTTP protocol observation.

Key commands used on R1:
```
interface GigabitEthernet0/0/0
 ip address 192.168.1.1 255.255.255.0
 no shutdown
```

Key commands used on S1:
```
interface vlan 1
 ip address 192.168.1.2 255.255.255.0
 no shutdown
ip default-gateway 192.168.1.1
```

---

## Part 2 — Connectivity Verification

All devices verified with `ping` and `arp -a`.

| Test | From | To | Result | TTL |
|---|---|---|---|---|
| PC1 → PC2 | 192.168.1.10 | 192.168.1.11 | ✅ 4/4 | 128 |
| PC2 → PC1 | 192.168.1.11 | 192.168.1.10 | ✅ 4/4 | 128 |

ARP tables on both PCs showed dynamic entries for R1 (`192.168.1.1`), S1 (`192.168.1.2`), and the opposite PC.

S1 MAC address table confirmed dynamic learning on Fa0/1 and Fa0/2.

---

## Part 3 — Encapsulation Analysis

Observed the full protocol stack during a PC1 → PC2 ping using simulation mode.

| Step | Protocol | OSI Layer | TCP/IP Layer | Key Field Observed |
|---|---|---|---|---|
| 1 | ARP Request | Layer 2 | Network Access | Target IP: 192.168.1.11 |
| 2 | ARP Reply | Layer 2 | Network Access | MAC: 000D.BDC4.3914 |
| 3 | ICMP Echo | Layer 3 | Internet | Type: 8, Code: 0 |
| 4 | Ethernet Frame | Layer 2 | Network Access | Src: 0060.2F28.0395 → Dst: 000D.BDC4.3914 |
| 5 | ICMP Reply | Layer 3 | Internet | Type: 0, Code: 0 |

**Encapsulation chain:**
```
Data → ICMP Header → IP Header (192.168.1.10/11) → Ethernet Header (MACs) → Bits
```

---

## Part 4 — Troubleshooting Challenge

Three deliberate faults were injected and resolved.

### Scenario A — Wrong Subnet Mask
- **Fault:** PC1 subnet mask changed to `255.255.0.0`
- **Symptom:** Ping to PC2 still succeeded (both in same /16 range), but ARP table only showed PC2 — not R1 or S1
- **Fix:** Restore subnet mask to `255.255.255.0`

### Scenario B — Router Interface Shutdown
- **Fault:** `shutdown` applied to R1 G0/0/0
- **Symptom:** `ping 192.168.1.1` → 100% loss; `arp -a` showed `No ARP Entries Found`
- **Fix:** `no shutdown` on R1 G0/0/0 — interface came back up, ARP resolved

### Scenario C — Wrong Default Gateway
- **Fault:** PC2 default gateway changed to `192.168.1.99` (non-existent)
- **Symptom:** First ping to R1 showed 25% loss; `route print` showed the incorrect gateway
- **Fix:** Reset default gateway to `192.168.1.1`

---

## Part 5 — Advanced Observation

### ARP Cache Behavior
Cleared ARP cache with `arp -d`, then re-pinged PC2. First reply took 475ms (ARP resolution delay); subsequent replies dropped back to <1ms once MAC was cached.

### MAC Address Table Learning
Cleared S1 MAC table with `clear mac address-table dynamic` — table showed empty. After a single PC1→PC2 ping, the table repopulated with all three MAC addresses dynamically.

### Protocol Interaction Flow

```
[PC1]
  1. Application: Echo Request command
  2. Transport: ICMP (no TCP/UDP needed)
  3. Network: IP Header — Src 192.168.1.10, Dst 192.168.1.11
  4. Data Link: Ethernet Frame — Src MAC 0060.2F28.0395
  5. Physical: Signal out FastEthernet0

[Switch0]
  Receive bits → Check Dst MAC → Forward out correct port

[PC2]
  Physical → Strip Ethernet → Strip IP → Read ICMP → Reply
```

---

## Lab Completion Checklist

| Checkpoint | Command | Expected | Result |
|---|---|---|---|
| R1 Interface Status | `show ip int brief` | G0/0 UP/UP | ✅ PASS |
| S1 VLAN Config | `show vlan brief` | VLAN 1 Active | ✅ PASS |
| PC1 Connectivity | `ping 192.168.1.11` | 4 replies | ✅ PASS |
| ARP Resolution | `arp -a` on PC1 | 2+ entries | ✅ PASS |
| MAC Learning | `show mac address-table` on S1 | 2 dynamic entries | ✅ PASS |

---

## Assessment Answers

1. **Which protocol resolves MAC before ICMP operates?** ARP — PC1 knows PC2's IP but needs its MAC address before Layer 2 framing can occur.
2. **What happens with subnet mask `255.255.255.128`?** Both PCs remain in the same /25 subnet so pings still work.
3. **Why doesn't the switch need an IP for basic forwarding?** Switches operate at Layer 2 using MAC address tables — IP is only needed for management access.
4. **Which OSI layers does R1 use to forward a packet?** Layers 1, 2, and 3.
5. **Purpose of TTL?** Decremented by 1 at each router hop. When TTL reaches 0 the packet is discarded, preventing infinite loops in the network.

---

## Files
- `Lab_1__Complete_Lab-Focused_Teaching_Package.pdf` — Full lab with screenshots
- `ITN_Module3_ProtocolLab.pkt` — Packet Tracer simulation file
