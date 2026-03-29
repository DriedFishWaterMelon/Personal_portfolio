# Assignment 4 — Network Connectivity Testing with Ping

**Course:** Network Architecture  
**Student:** Achirawat Buengsai `673380298-3`  
**Tool:** Cisco Packet Tracer — PC Command Line

---

## Overview

This assignment verifies end-to-end network connectivity across a multi-subnet network using the `ping` command. Tests are performed from two different PCs, each targeting addresses both within and across subnet boundaries.

---

## Test Results

### PC 1 — Tests from `192.168.1.x` subnet

| Destination | Result | Packets | Loss | Avg RTT |
|---|---|---|---|---|
| `192.168.1.3` | ✅ Success | 4/4 | 0% | ~0ms |
| `192.168.2.1` | ✅ Success | 4/4 | 0% | ~5ms |
| `192.168.1.1` | ✅ Success | 4/4 | 0% | ~0ms |

**Observations:**
- Ping to `192.168.1.3` (same subnet) responds with `TTL=128` and sub-millisecond RTT — traffic stays within the local LAN, no routing needed
- Ping to `192.168.2.1` (different subnet) responds with `TTL=254` and slightly higher RTT (~5ms average) — traffic crosses through the **router**, which decrements the TTL

---

### PC 2 — Tests from `192.168.2.x` subnet

| Destination | Result | Packets | Loss | Avg RTT |
|---|---|---|---|---|
| `192.168.2.1` | ✅ Success | 4/4 | 0% | ~0ms |
| `192.168.1.1` | ✅ Success | 4/4 | 0% | ~23ms avg |

**Observations:**
- Ping to `192.168.2.1` (same subnet) — `TTL=255`, ~0ms, no routing required
- Ping to `192.168.1.1` (different subnet) — `TTL=254`, first packet shows high latency (87ms) due to ARP resolution; subsequent packets drop to 1ms

---

## Key Concepts Demonstrated

**TTL (Time To Live)** decreases by 1 each time a packet passes through a router:
- TTL=128 or 255 → same subnet (no router hop)
- TTL=254 → one router hop crossed

**First-packet latency** is higher when ARP must resolve an unknown MAC address before the packet can be forwarded — subsequent packets use the cached ARP entry and respond much faster.

**0% packet loss** across all tests confirms the network topology is correctly configured with working routing between the two subnets.

---

## Network Setup (Inferred)

```
[PC - 192.168.1.x subnet] ── Switch ──┐
                                       Router (192.168.1.1 / 192.168.2.1)
[PC - 192.168.2.x subnet] ── Switch ──┘
```

---

## Files
- `assign_4.pdf` — Ping command output screenshots
