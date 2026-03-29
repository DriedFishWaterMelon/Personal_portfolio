# Lab 4 — Simulated Internet (10.10.0.0/16) & Private LANs with Stateful vs Stateless Services

**Course:** Network Architecture  
**Tools:** Cisco Packet Tracer, Python 3, VS Code, Podman/Docker  
**File:** `Lab4.pkt`  
**Repository:** `automation/`

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

This lab simulates a real-world two-site WAN architecture. Two private LANs (Site A and Site B) are connected through a simulated public internet (`10.10.12.0/30`). Each site has a router performing **NAT overload (PAT)** and **static routing**, and each runs a mockup infrastructure deploying both a **stateless API** (port 3000) and a **stateful API** (port 3001). The lab compares session behavior for locally-accessed vs remotely-accessed services across the simulated internet.

---

## Network Topology

```
              Public / Internet
              10.10.12.0/30
         ┌──────────────────────┐
         │                      │
    R1 (10.10.12.1)        R2 (10.10.12.2)
    G0/1: 192.168.10.1     G0/1: 192.168.20.1
         │                      │
      SwitchA                SwitchB
      /       \              /       \
 ServerA    ClientA      ServerB    ClientB
192.168.10.10  .50    192.168.20.10   .50
```

---

## Address Table

| Component | Interface | IP | Mask | Role |
|---|---|---|---|---|
| R1 | G0/0 | 10.10.12.1 | /30 | WAN (NAT outside) |
| R1 | G0/1 | 192.168.10.1 | /24 | LAN A Gateway (NAT inside) |
| ServerA | eth0 | 192.168.10.10 | /24 | Mockup infra — Site A |
| ClientA | eth0 | 192.168.10.50 | /24 | Tester — Site A |
| R2 | G0/0 | 10.10.12.2 | /30 | WAN (NAT outside) |
| R2 | G0/1 | 192.168.20.1 | /24 | LAN B Gateway (NAT inside) |
| ServerB | eth0 | 192.168.20.10 | /24 | Mockup infra — Site B |
| ClientB | eth0 | 192.168.20.50 | /24 | Tester — Site B |

---

## Part 1 — Cisco Network Configuration

### R1 Configuration (LAN A)

```
interface g0/0
 ip address 10.10.12.1 255.255.255.252
 ip nat outside
 no shutdown

interface g0/1
 ip address 192.168.10.1 255.255.255.0
 ip nat inside
 no shutdown

ip route 192.168.20.0 255.255.255.0 10.10.12.2
access-list 10 permit 192.168.10.0 0.0.0.255
ip nat inside source list 10 interface g0/0 overload
```

**What R1 does:**
- `g0/1` — LAN A internal interface (NAT inside)
- `g0/0` — WAN interface toward R2 (NAT outside)
- Static route sends all LAN B traffic (`192.168.20.0/24`) via next-hop R2 (`10.10.12.2`)
- NAT overload (PAT) translates all LAN A addresses to R1's public IP, distinguished by port number

### R2 Configuration (LAN B)

```
interface g0/0
 ip address 10.10.12.2 255.255.255.252
 ip nat outside
 no shutdown

interface g0/1
 ip address 192.168.20.1 255.255.255.0
 ip nat inside
 no shutdown

ip route 192.168.10.0 255.255.255.0 10.10.12.1
access-list 10 permit 192.168.20.0 0.0.0.255
ip nat inside source list 10 interface g0/0 overload
```

R2 is a mirror of R1 — same structure, opposite direction.

---

## Routing & NAT Verification

### show ip route (R1)
```
C    192.168.10.0/24 is directly connected, g0/1
C    10.10.12.0/30   is directly connected, g0/0
S    192.168.20.0/24 [1/0] via 10.10.12.2
```

### show ip route (R2)
```
C    192.168.20.0/24 is directly connected, g0/1
C    10.10.12.0/30   is directly connected, g0/0
S    192.168.10.0/24 [1/0] via 10.10.12.1
```

Bidirectional static routing is complete — both sites can reach each other.

### show ip nat translations (R1)
```
Pro  Inside local        Inside global    Outside global
icmp 192.168.10.10:1    10.10.12.1:1     192.168.20.10:1
```

PAT is working — ClientA's private IP is translated to R1's public interface IP.

### Cross-site ping (from ClientA)
```
ping 10.10.12.2  →  !!!!!  (100% success, TTL=254)
```

---

## Part 2 — IaC / Mockup Infrastructure Deployment

### Repository Structure
```
automation/
├── mockup-infra/
└── week02-stateless-stateful/
    └── phase1-mockup/
```

### Environment Configuration

**ServerA.env**
```env
PUBLIC_NET_SUBNET=192.168.10.0/24
HOST_IP=192.168.10.10
REMOTE_NET_PUBLIC=10.10.12.0/30
```

**ServerB.env**
```env
PUBLIC_NET_SUBNET=192.168.20.0/24
HOST_IP=192.168.20.10
REMOTE_NET_PUBLIC=10.10.12.0/30
```

### Deploy (Windows PowerShell)
```powershell
cd mockup-infra
python manage.py init
python manage.py deploy --profile week02-stateless-stateful
```

### Deploy (Linux/Bash)
```bash
cd mockup-infra
chmod +x scripts/*.sh
./scripts/init-env.sh
python manage.py deploy --profile week02-stateless-stateful
```

`python manage.py init` automatically generates TLS certificates, creates isolated virtual networks (private/public), and confirms with `Infrastructure initialized successfully`.

### Running Containers

| Container | Port | Type |
|---|---|---|
| stateless-api | 3000 | No session state |
| stateful-api | 3001 | Session-aware |
| mockup-gateway | 80/443 | Nginx reverse proxy |

---

## Part 3 — Stateless vs Stateful Testing

### Local Tests (Private LAN A)

**Stateless:**
```bash
curl http://192.168.10.10:3000/info
# → {"status":"success","message":"Stateless server","data":{...},"requestCount":1}
```
Every request is independent — no memory of previous calls.

**Stateful:**
```bash
curl -X POST http://192.168.10.10:3001/session
# → Returns Session-ID

curl http://192.168.10.10:3001/data -H "Session-ID: <id>"
# → Returns data scoped to that session
```

### Remote Tests (Via Simulated Internet / NAT)

```bash
# Target ServerB's stateless API from ClientA
curl http://10.10.12.2:3000/info
# → Routed through NAT + static routes → ServerB responds

# Stateful session across NAT
curl -X POST http://10.10.12.2:3001/session
curl http://10.10.12.2:3001/data -H "Session-ID: <id>"
# → Session ID persists across NAT boundary (logical identity survives PAT)
```

### Network Isolation Proof
- Accessing `localhost:8080` from Windows (outside the container network) → `Unable to connect`
- Accessing from inside the gateway container via `podman exec` → `HTTP 200 OK`

This confirms the system correctly implements "outsiders cannot reach in, insiders can communicate."

---

## Troubleshooting

| Symptom | Likely Cause | Fix |
|---|---|---|
| Cannot ping remote LAN | Wrong or missing static route | `show ip route` — verify S entries exist |
| NAT not translating | ACL mismatch or wrong `ip nat inside/outside` | Check `show ip nat translations` |
| Session lost between requests | Missing `Session-ID` header | Include header on all stateful requests |
| Cannot reach port 3001 | OS firewall or container port mapping wrong | Check `podman ps` and container ports |
| `ping 10.10.12.2` fails from R1 | Interface not up | `show ip interface brief` — check G0/0 status |

---

## Engineering Reflection

1. **Stateless services** treat every request as new — no prior context required. Horizontally scalable, no session affinity needed.

2. **Stateful services** maintain session context across requests — the client must send a Session-ID header. More complex to scale but necessary for personalized or multi-step workflows.

3. **NAT does not break logical session identity.** PAT translates IP:port pairs but the Session-ID in the HTTP header is untouched at Layer 3 — the application layer remains consistent.

4. **Static routing** is sufficient for a two-site lab but would not scale. OSPF would automatically propagate routes as topology changes — a natural extension of this lab.

---

## Files
- `LAB_4_Simulated_Internet__10_10_0_0_16____Private_LANs_with_Stateful_vs_Stateless_Services.pdf` — Full lab report
- `Lab4.pkt` — Packet Tracer simulation file
- `automation/mockup-infra/` — Infrastructure as Code deployment scripts
- `automation/week02-stateless-stateful/phase1-mockup/` — Stateless & stateful API source
