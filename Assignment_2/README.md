# Assignment 2 — Network Topology Design in Cisco Packet Tracer

**Course:** Network Architecture  
**Student:** Achirawat Buengsai `673380298-3`  
**Tool:** Cisco Packet Tracer

---

## Overview

This assignment demonstrates three progressively scaled network topologies — from the smallest possible two-device connection up to a multi-subnet hierarchical network — built and verified in Cisco Packet Tracer.

---

## Topology 1 — Peer-to-Peer (P2P) Network

**Devices:** 2 × PC (PC0, PC1)  
**Cable:** Copper Cross-Over Cable (dashed black line)  
**Status:** Green triangles at both ends → Link Up ✅

The simplest possible network. Two computers connected directly without any intermediary device. Data flows directly between machines on the same cable.

> **Answer: The smallest network** is a Peer-to-Peer (P2P) connection — just 2 computers linked directly by a crossover cable.

---

## Topology 2 — LAN with Switch

**Devices:** 3 × PC (PC2, PC3, PC4) + 1 × Switch (Cisco 2960)  
**Cable:** Copper Straight-Through Cable (solid black line) — used for different device types (PC → Switch)  
**Status:** Green triangles at all ports → Link Up ✅

A Local Area Network where the switch acts as the central hub, allowing all three PCs to communicate with each other through a shared fabric.

> **Answer: A network requiring a Switch** is a LAN — needed when connecting more than 2 devices within the same network segment.

---

## Topology 3 — Hierarchical / Tree Topology (Inter-Network)

**Devices:**
- 1 × Router (Cisco ISR4321)
- 2 × Switch (Cisco 2960) — one per subnet
- 6 × PC — 3 per switch (PC2(1), PC3(1), PC4(1) | PC2(2), PC3(2), PC4(2))

**Cable:** Copper Straight-Through (all connections)  
**Status:** Green triangles at all connection points → Link Up ✅

A two-LAN hierarchical network where the router connects two separate subnets. Each switch distributes connectivity to its local PCs, while the router handles inter-network routing.

> **Answer: A network requiring a Router** is any inter-network (WAN/multi-subnet) setup — when two or more LAN segments need to communicate across different subnets, a router is required to forward traffic between them. A standard switch cannot route across subnets.

---

## Cable Type Reference

| Cable Type | Visual | Use Case |
|---|---|---|
| Copper Cross-Over | Dashed line | Same device type (PC ↔ PC) |
| Copper Straight-Through | Solid line | Different device types (PC ↔ Switch, Switch ↔ Router) |

---

## Summary

| Topology | Devices Needed | Scale |
|---|---|---|
| Peer-to-Peer | 2 PCs + crossover cable | Smallest — no intermediary |
| LAN | PCs + Switch | Single subnet, multiple devices |
| Hierarchical/WAN | PCs + Switches + Router | Multiple subnets / inter-network |

---

## Files
- `Assignment_2.pdf` — Topology diagrams and explanations
