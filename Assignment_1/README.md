# Assignment 1 — Observe Network Traffic & TCP/UDP Protocol Analysis

**Course:** Network Architecture  
**Student:** Achirawat Buengsai `673380298-3`  
**Tool:** Cisco Packet Tracer (Simulation Mode)

---

## Overview

This lab explores how multiple types of network traffic coexist on a shared network using **multiplexing**, and examines the behavioral differences between **TCP** (reliable) and **UDP** (unreliable) transport layer protocols through live packet capture simulation.

---

## Network Topology

A star topology with a central **Switch** connecting:
- **MultiServer** (`192.168.1.254`) — hosts HTTP, FTP, DNS, SMTP/POP3 services
- **HTTP Client** (`192.168.1.1`)
- **FTP Client**
- **DNS Client** (`192.168.1.3`)
- **E-Mail Client** (`192.168.1.4`)

---

## Part 1 — Multiplexing

Traffic types generated simultaneously: ARP, HTTP, FTP, DNS, and Email.

**Key findings:**
- Multiple PDUs from different hosts travel the network at the same time — this is called **conversation multiplexing**
- Only **one PDU** can cross a wire in each direction at any given time
- Different colors in the Simulation Panel event list represent **different protocols**

---

## Part 2 — TCP vs UDP Protocol Behavior

### HTTP (TCP — Port 80)
| Field | Value |
|---|---|
| SRC PORT | 1026 |
| DEST PORT | 80 |
| Initial SEQ NUM | 0 |
| Initial ACK NUM | 0 |
| Flags | SYN → SYN+ACK → PSH+ACK |

- HTTP takes longer to appear because **TCP must complete a 3-way handshake** before data transfer begins
- Communications are **reliable** — TCP guarantees delivery and ordering

### FTP (TCP — Port 21)
| Field | Value |
|---|---|
| SRC PORT | 1026 |
| DEST PORT | 21 |
| Initial Flags | SYN |
| Server Response Flags | SYN+ACK |
| Final ACK Flags | ACK (SEQ=1, ACK=1) |

- Server welcome message: `"Welcome to PT Ftp server"`
- Like HTTP, FTP uses TCP for **reliable, connection-oriented** transfer

### DNS (UDP — Port 53)
| Field | Value |
|---|---|
| SRC PORT | 1026 |
| DEST PORT | 53 |
| SEQ / ACK NUM | None |

- Layer 4 protocol: **UDP**
- No sequence/acknowledgement numbers — **UDP does not establish a connection**
- Communications are **not reliable** — no delivery guarantee
- DNS response resolves `multiserver.pt.ptu` → `192.168.1.254` via the **DNS ANSWER** section

### Email — SMTP/POP3 (TCP — Ports 25 / 110)
| Protocol | Port |
|---|---|
| SMTP (Send) | 25 |
| POP3 (Receive) | 110 |

- Email uses **TCP** — reliable, connection-oriented
- Same 3-way handshake pattern as HTTP and FTP (SYN → SYN+ACK → ACK → PSH+ACK for data)

---

## Key Concepts Demonstrated

| Protocol | Transport | Reliable | Port |
|---|---|---|---|
| HTTP | TCP | ✅ Yes | 80 |
| FTP | TCP | ✅ Yes | 21 |
| SMTP | TCP | ✅ Yes | 25 |
| POP3 | TCP | ✅ Yes | 110 |
| DNS | UDP | ❌ No | 53 |

- **TCP** uses a **3-way handshake** (SYN → SYN+ACK → ACK) before sending application data
- **UDP** sends data immediately with no handshake — faster but without delivery guarantees
- Port numbers identify services; source/destination ports **swap** on the server's reply

---

## Files
- `assign_4.pdf` — Lab screenshots and answers
