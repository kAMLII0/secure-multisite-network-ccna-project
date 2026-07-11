
# Secure Multi-Site Enterprise Network with Redundant Routing

## Overview
This project simulates a two-site enterprise network (Headquarters and Branch office) connected over a simulated ISP, built in Cisco Packet Tracer. It demonstrates dynamic routing, first-hop gateway redundancy, NAT-based internet access, inter-site traffic segmentation, and Layer 2 attack prevention — combining core routing/switching concepts with practical network security hardening.

**Tools used:** Cisco Packet Tracer
**How to test:** Click `project.2.pkt` in this repository, then click **Download** to download the `.pkt` file and open it in Cisco Packet Tracer.

## Objectives
- Connect HQ and Branch sites using dynamic routing (OSPF), eliminating static route dependencies
- Eliminate the core switch as a single point of failure using HSRP gateway redundancy
- Provide internet access to internal hosts via NAT/PAT, while giving a public-facing server a fixed address via static NAT
- Restrict Branch-to-HQ traffic to only explicitly permitted services using extended ACLs
- Harden the access layer against DHCP starvation and ARP spoofing attacks using DHCP Snooping and Dynamic ARP Inspection (DAI)

## Topology

![Topology Diagram])

| Device | Role | Model |
|---|---|---|
| core-SW1 | HQ Core (Active HSRP, routes VLANs 10/20/99) | 3650-24PS |
| core-SW2 | HQ Core (Standby HSRP) | 3650-24PS |
| Access-SW1 | HQ Access Layer (DHCP Snooping, DAI) | 2960-24TT |
| HQ-Edge-RTR | HQ Edge Router (OSPF, NAT, ACL) | 2911 |
| Branch-RTR | Branch Router | 1941 |
| Branch-SW1 | Branch Access Switch | 2960-24TT |
| ISP-Router | Simulated Internet/ISP | 2811 |
| WEB-Server, DHCP-Server | HQ Servers (VLAN 20) | Server-PT |
| PC1, PC2 | HQ Employee hosts (VLAN 10) | PC-PT |
| branch-pc1 | Branch host | PC-PT |


## IP Addressing

| Segment | Subnet | Gateway |
|---|---|---|
| VLAN 10 — Employees | 192.168.10.0/24 | 192.168.10.254 (HSRP VIP) |
| VLAN 20 — Servers | 192.168.20.0/24 | 192.168.20.254 (HSRP VIP) |
| VLAN 99 — Guest | 192.168.99.0/24 | 192.168.99.254 (HSRP VIP) |
| Branch LAN | 192.168.30.0/24 | 192.168.30.1 |
| HQ–ISP WAN | 203.0.113.0/30 | — |
| HQ–Core transit | 10.0.0.0/30 | — |
| HQ–Branch WAN | 10.0.0.4/30 | — |



## Key Configurations

**Inter-VLAN Routing & HSRP (Core Switches)**
- SVIs created for VLANs 10, 20, 99 on both core switches
- HSRP configured per VLAN: core-SW1 priority 110 (Active), core-SW2 priority 90 (Standby), both with `preempt` enabled

**Dynamic Routing**
- OSPF Area 0 running between core-SW1, HQ-Edge-RTR, and Branch-RTR
- `default-information originate` on HQ-Edge-RTR to propagate the internet-bound default route to Branch

**NAT**
- Static NAT: WEB-Server (192.168.20.10) mapped to public IP 203.0.113.10
- Dynamic NAT with PAT (`overload`) for general internal client internet access

**Access Control**
- Extended ACL on HQ-Edge-RTR restricting Branch traffic: permits Branch → WEB-Server (port 80 only), denies Branch → Employee VLAN, permits all other traffic

**Layer 2 Security (Access-SW1)**
- DHCP Snooping enabled on VLANs 10/20/99, uplinks marked trusted, rate-limiting applied on untrusted access ports
- Dynamic ARP Inspection enabled on VLANs 10/20/99, validated against the DHCP snooping binding table

---

## Verification / Test Cases

| # | Test | Command / Action | Expected Result |
|---|---|---|---|
| 1 | HQ gateway reachability | `ping 192.168.10.254` from PC1 | Success |
| 2 | Inter-VLAN routing | `ping 192.168.20.10` from PC1 | Success |
| 3 | HSRP failover | Shut `interface vlan 10` on core-SW1 | core-SW2 becomes Active within seconds; pings to `.254` continue |
| 4 | Cross-site routing (OSPF) | `ping 192.168.30.10` from PC1 | Success |
| 5 | ACL — permitted path | `ping 192.168.20.10` from branch-pc1 |
