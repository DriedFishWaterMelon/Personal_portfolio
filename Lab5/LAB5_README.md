# 🧪 LAB 5 — Internet Edge + ISP Serial WAN + Week03 Microservices

> **Course:** CP352005 Networks  
> **Type:** Practical Lab Assignment  
> **Topic:** Enterprise WAN Architecture with NAT, Serial ISP Link, and Distributed Microservices

---

## 👥 Team Members

| Name | Student ID |
|------|-----------|
| เพชรภิญโญ ธนศิรินรากร | 673380073-7 |
| ก้องภพ โชควิริยะ | 673380030-5 |
| ณพวิทย์ วงษ์ประเสริฐ | 673380266-6 |
| รัฐภูมิ เกิดพระจีน | 673380057-5 |
| อชิรวัช บึงไสย์ | 673380298-3 |

---

## 🎯 Objective

Transform Week 03 microservices into a realistic enterprise deployment by building a full WAN topology featuring:

- **R1** as Internet Edge Router (NAT + DHCP WAN)
- **Serial WAN** as the ISP Domain (`100.10.10.0/30`)
- **R2** as Branch Router
- **ServerA & ServerB** running distributed Week03 microservices
- Cross-site microservice access validated over routed WAN + NAT

---

## 🗺️ Network Topology

```
              INTERNET
                 |
              (DHCP)
           R1 G0/0 [NAT OUTSIDE]
                 |
           R1 S0/0/0  ←── 100.10.10.1/30
                 |
         [ISP Serial WAN]
                 |
           R2 S0/0/0  ←── 100.10.10.2/30
          /              \
   R1 G0/1           R2 G0/1
192.168.10.1       192.168.20.1
     |                    |
  ServerA             ServerB
192.168.10.10      192.168.20.10
(Microservices A)  (Microservices B)
```

---

## 📍 Address Table

| Device  | Interface | IP Address       | Role            |
|---------|-----------|------------------|-----------------|
| R1      | G0/0      | DHCP             | Internet (WAN)  |
| R1      | S0/0/0    | 100.10.10.1/30   | ISP Serial Link |
| R1      | G0/1      | 192.168.10.1/24  | LAN A Gateway   |
| R2      | S0/0/0    | 100.10.10.2/30   | ISP Serial Link |
| R2      | G0/1      | 192.168.20.1/24  | LAN B Gateway   |
| ServerA | eth0      | 192.168.10.10    | Microservices A |
| ServerB | eth0      | 192.168.20.10    | Microservices B |

---

## 🔧 Part 1 — Cisco Router Configuration

### R1 — Internet Edge Router

```cisco
enable
conf t
hostname R1

interface g0/0
 ip address dhcp
 ip nat outside
 no shutdown

interface g0/1
 ip address 192.168.10.1 255.255.255.0
 ip nat inside
 no shutdown

interface s0/0/0
 ip address 100.10.10.1 255.255.255.252
 clock rate 64000
 no shutdown

ip route 192.168.20.0 255.255.255.0 100.10.10.2
ip route 0.0.0.0 0.0.0.0 g0/0

access-list 10 permit 192.168.10.0 0.0.0.255
ip nat inside source list 10 interface g0/0 overload

end
write memory
```

### R2 — Branch Router

```cisco
enable
conf t
hostname R2

interface g0/1
 ip address 192.168.20.1 255.255.255.0
 no shutdown

interface s0/0/0
 ip address 100.10.10.2 255.255.255.252
 no shutdown

ip route 192.168.10.0 255.255.255.0 100.10.10.1
ip route 0.0.0.0 0.0.0.0 100.10.10.1

end
write memory
```

---

## 💻 Part 2 — Microservices Deployment

Deploy Week03 microservices on both servers using the Phase 1 architecture.

### On ServerA (LAN A — 192.168.10.10)

```bash
cd automation/week03-microservices/phase1
python start_services.py
```

| Service    | Port |
|------------|------|
| Upload     | 8000 |
| Processing | 8001 |
| AI         | 8002 |
| Gateway    | 9000 |

### On ServerB (LAN B — 192.168.20.10)

```bash
cd automation/week03-microservices/phase1
python start_services.py
```

Same service layout as ServerA.

---

## 🧪 Part 3 — WAN & Internet Testing

### Test 1 — Cross-Site Health Check (LAN A → LAN B)

```bash
# From ClientA
curl http://192.168.20.10:8000/health
```

**Expected path:** `ClientA → R1 → Serial ISP → R2 → ServerB`

### Test 2 — Full Workflow Across ISP

```bash
# From LAN A client
curl -X POST http://192.168.20.10:9000/process-file
```

**Flow:** `Upload → Processing → AI → Response returned across WAN`

### Test 3 — Internet Reachability via NAT

```bash
# From ServerA
ping 8.8.8.8
```

**Verify NAT on R1:**
```cisco
show ip nat translations
```

---

## 🔎 Verification Commands

**On R1:**
```cisco
show ip route
show ip nat translations
show ip interface brief
```

**On R2:**
```cisco
show ip route
ping 100.10.10.1
```

---

## 🧠 Engineering Concepts Demonstrated

| Concept | Where Demonstrated |
|---|---|
| Public vs Private Boundaries | R1 NAT Edge |
| Distributed Services | ServerA & ServerB |
| Network Segmentation | Serial ISP `/30` |
| Horizontal Architecture | Two independent microservice stacks |
| Internet Edge Pattern | R1 as NAT Gateway |
| Service Isolation | LAN A vs LAN B |

---

## 🔬 What Changed from Lab 4

| Lab 4 | Lab 5 |
|-------|-------|
| `10.10.12.0/30` simulated internet | Real ISP serial `100.10.10.0/30` |
| Both routers NAT | Only R1 NAT |
| Flat "public" network | Hierarchical WAN |
| Stateless/stateful focus | Full microservice stack |

---

## 🏗️ Architectural Significance

This lab implements a **miniature cloud-edge + branch distributed architecture** featuring:

- Enterprise Edge Router with NAT boundary
- ISP Serial WAN link
- Branch Router with static routing
- Dual microservice deployments across network segments
- Cross-site distributed system communication

> *This is no longer a classroom NAT lab — it's a production-style enterprise WAN in miniature.*
