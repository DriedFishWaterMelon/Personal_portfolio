# Lab 3 — MIME File Transfer over Router-on-a-Stick with Wireshark Analysis

**Course:** Network Architecture  
**Tools:** Cisco Packet Tracer, Python 3, Wireshark  
**File:** `lab3.pkt`

---

## Team

| Name | Student ID |
|---|---|
| เพชรภิญโญ ธนศิรินรากร | 673380073-7 |
| ก้องภพ โชควิริยะ | 673380030-5 |
| ณพวิทย์ วงษ์ประเสริฐ | 673380266-6 |
| รัฐภูมิ เกิดพระจีน | 673380057-5 |
| อชิรวัช บึงไสย์ | 673380298-3 |

---

## Overview

This lab bridges classical network configuration with application-layer engineering. A MIME-aware Python socket server is deployed on a server in VLAN 20, while a client in VLAN 10 transfers files to it across a Router-on-a-Stick inter-VLAN boundary. Wireshark captures the full protocol stack — from ARP resolution through the TCP handshake to the JSON MIME header in the payload — demonstrating how application meaning survives multiple layers of encapsulation and routing.

---

## Scenario

> A client in VLAN 10 transfers a file to a server in VLAN 20 using a custom MIME-based protocol (JSON header + binary payload). The lab verifies connectivity, observes encapsulation at each layer, and captures the transfer in Wireshark.

---

## Network Topology

```
PC1 (VLAN 10, 192.168.10.10) ──\
                                 \── [S1] ──trunk── [R1]
                                 /
Server0 (VLAN 20, 192.168.20.20) ──/
```

---

## Addressing Scheme

| Device | Interface | IP Address | Subnet Mask | Gateway |
|---|---|---|---|---|
| R1 | G0/0.10 | 192.168.10.1 | 255.255.255.0 | — |
| R1 | G0/0.20 | 192.168.20.1 | 255.255.255.0 | — |
| PC1 (Client) | NIC | 192.168.10.10 | 255.255.255.0 | 192.168.10.1 |
| Server0 (Server) | NIC | 192.168.20.20 | 255.255.255.0 | 192.168.20.1 |

---

## Part 1 — Network Device Configuration

### Router (R1)
```
interface g0/0
 no shutdown

interface g0/0.10
 encapsulation dot1Q 10
 ip address 192.168.10.1 255.255.255.0

interface g0/0.20
 encapsulation dot1Q 20
 ip address 192.168.20.1 255.255.255.0
```

### Switch (S1)
```
vlan 10
 name CLIENTS
vlan 20
 name SERVERS

interface g0/1
 switchport mode trunk

interface fa0/1
 switchport mode access
 switchport access vlan 10

interface fa0/2
 switchport mode access
 switchport access vlan 20
```

---

## Part 2 — Connectivity Verification

From PC1:
```
ping 192.168.20.20   → ✅ Successful replies
```

Verified on R1:
```
show ip route         → Both subnets directly connected
show ip interface brief → Both subinterfaces up/up
```

Verified on S1:
```
show interfaces trunk → Gig0/1 trunking, 802.1q, VLANs 10 & 20 active
show mac address-table → Dynamic entries populated after ping
```

> Note: Cisco Packet Tracer does not support a PC as a socket server — Server-PT (Server0) was used instead, with the web browser simulating the JSON response.

---

## Part 3 — MIME Server Deployment

### Server (PC2 / Server0)
```bash
cd automation
python server/main_enhanced.py --host 0.0.0.0 --port 65432 --verbose
```
Output:
```
[15:35:13] INFO: Starting server with host=0.0.0.0, port=65432, storage=storage
[15:35:13] INFO: Server listening on 0.0.0.0:65432
```

### Client (PC1)
```bash
cd automation
python client/main_enhanced.py assets/sample.png --host 192.168.20.20 --port 65432
```
Output:
```
[15:38:10] INFO: Connected to 127.0.0.1:65432
[15:38:10] INFO: Sent assets/sample.png (108 bytes) as image/png
[15:38:10] INFO: Connection closed. Sent: 1, Failed: 0
[15:38:10] INFO: All files sent successfully
```

The file was saved on the server, confirming end-to-end delivery.

---

## Part 4 — Wireshark Capture Analysis

Capture filter used: `tcp port 65432`

### A — ARP Exchange
Before TCP can begin, PC1 ARPs for its gateway (`192.168.10.1`):
- ARP Request: "Who has 192.168.10.1?"
- ARP Reply: R1 responds with its MAC address

### B — TCP Three-Way Handshake
Observed in PDU details (Source Port: 1030, Destination Port: 80):

| Step | Flag | Src Port | Dst Port | SEQ | ACK |
|---|---|---|---|---|---|
| 1 | SYN | 1030 | 80 | 0 | 0 |
| 2 | SYN-ACK | 80 | 1030 | 0 | 1 |
| 3 | ACK | 1030 | 80 | 1 | 1 |

### C — Data Transfer (MIME Payload)
Following the TCP stream revealed the JSON header prepended to the binary payload:
```json
{"mime_type": "image/png", "size": 12345}
```
Followed immediately by raw binary file data.

### D — VLAN Tags (802.1Q)
When capturing on the trunk interface, 802.1Q tags were visible in the Ethernet frame with the correct VLAN ID embedded.

---

## Packet Dissection Guide

| Layer | Field | What to Observe |
|---|---|---|
| Ethernet | Src/Dst MAC | Frame direction |
| 802.1Q | VLAN ID | Traffic segmentation |
| IP | TTL | Routing behavior (decrements at each hop) |
| TCP | Flags | Reliability — SYN, ACK, PSH, FIN |
| Payload | JSON | MIME metadata wrapping the binary file |

---

## Part 5 — Troubleshooting Scenarios

### Scenario 1 — Broken Routing (Subinterface Shutdown)
- **Fault:** R1 subinterface G0/0.10 shut down
- **Observation:** `ping 192.168.10.1` → 100% loss; TCP connection attempts time out
- **Fix:** `no shutdown` on G0/0.10

### Scenario 2 — Firewall Block
- **Fault:** Port 65432 blocked at server
- **Observation:** Client sends repeated SYN packets with no SYN-ACK response — TCP retransmission storm visible in Wireshark
- **Explanation:** When a port is blocked, the TCP handshake cannot complete. The client retransmits SYN up to its retry limit before giving up. This demonstrates that correct IP/routing configuration alone is not sufficient — application-layer port policies matter too.

### Scenario 3 — Wrong Gateway
- **Fault:** PC1 gateway set to a non-existent IP
- **Observation:** ARP table shows `incomplete` for the gateway entry — no ARP Reply is ever received, so Layer 2 framing cannot complete and no packets leave the subnet
- **Explanation:** Without a valid gateway MAC, the host cannot construct an Ethernet frame for cross-subnet delivery.

### Scenario 4 — Packet Loss
- **Observation:** TCP retransmissions appear in Wireshark; window size adjustments visible as flow control engages; congestion control reduces send rate
- **Explanation:** TCP's reliability mechanism detects missing ACKs and retransmits lost segments. The transfer completes correctly despite partial loss, demonstrating TCP's fundamental reliability guarantee.

---

## Protocol Interaction Summary

Each layer is blind to the layers above it — this is how application meaning survives the journey:

```
[PC1 sends "image/png" file]
  Layer 7: JSON MIME header + binary payload
  Layer 4: TCP segments — reliable delivery guaranteed
  Layer 3: IP packet — src 192.168.10.10, dst 192.168.20.20
  Layer 2: Ethernet frame — src/dst MAC (changes at each hop)
  Layer 1: Electrical signal

[R1 routes at Layer 3]
  Reads IP header only — does not touch MIME payload
  Strips old Ethernet frame, writes new one for VLAN 20

[Server0 receives]
  Strips Ethernet → strips IP → strips TCP → reads JSON header → saves file
```

The core payload (`{"mime_type":"image/png", ...}` + binary) was never modified in transit.

---

## Assessment Answers

1. **Why does TCP retransmit lost segments?** TCP is connection-oriented — if no ACK arrives within the timeout window, it assumes the segment was lost and resends it.

2. **How does VLAN tagging affect broadcast domains?** 802.1Q tags restrict broadcast forwarding to only ports within the same VLAN, preventing ARP and other broadcasts from flooding the entire network.

3. **What role does MIME play compared to TCP headers?** TCP is the envelope (handles delivery). MIME is the cover letter (tells the application what the contents are — image/png, text/html, etc.).

4. **Where does the routing decision occur?** R1 at Layer 3 — it reads the destination IP, looks up its routing table, and forwards the packet out the correct subinterface.

5. **Why does ARP precede TCP?** You cannot build an Ethernet frame without knowing the destination MAC address. ARP resolves the Layer 3 IP to a Layer 2 MAC before any TCP segment can be sent.

---

## Files
- `__Lab3___MIME_File_Transfer_over_Router-on-a-Stick_with_Wireshark_Analysis.pdf` — Full lab report
- `lab3.pkt` — Packet Tracer simulation file
- `automation/` — Python MIME server and client source code
