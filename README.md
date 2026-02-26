# Multi-Site Hybrid WAN: Dual-Cloud DMVPN over MPLS with Internet Failover

## 📌 Project Overview

This repository contains the configuration and design for a resilient Enterprise Network. The architecture is split into a high-availability LAN switching fabric and a redundant Hybrid WAN edge that utilizes both MPLS and Internet transport.

---

## 🌐 Part 1: WAN Architecture & Redundant Design

The WAN edge is engineered for maximum resilience, ensuring that even the failure of a primary Hub (NHS) or a physical ISP link does not result in downtime.

---

## 1.1 Dual-Cloud DMVPN over MPLS (Primary Path)

To eliminate the Hub as a single point of failure, I implemented a Dual-Cloud DMVPN Phase 3 architecture rather than a single-cloud dual-hub design. This ensures complete control-plane separation.

### Cloud 22 (Primary):

- Interface: Tunnel 22 on all CE routers.
- Addressing: 172.16.22.0/24.
- NHS: CE1 serves as the Next Hop Server.
- Routing: Runs OSPF Area 3. To prevent default route conflicts with the existing ISP default routes, this was configured as a Normal Area with Manual Summarization rather than a Total NSSA.

### Cloud 33 (Redundant):

- Interface: Tunnel 33.
- Addressing: 172.16.33.0/24.
- NHS: CE2 serves as the Next Hop Server.
- Routing: Runs OSPF Area 4.
- Purpose: If CE1 fails, the OSPF adjacency and NHRP mappings remain active via Cloud 33, ensuring uninterrupted site-to-site connectivity.

---

## 1.2 Multi-Layer Security

All WAN traffic is secured using a standardized encryption framework:

- Encryption: IPsec protects all GRE traffic across the MPLS and Internet.
- IKEv2 Framework: Used for the DMVPN clouds to take advantage of the Next-Generation Encryption (NGE) and the simplified exchange process of IKEv2.
- IKEv1 Legacy: Maintained for specific GRE-over-IPsec tunnels where legacy compatibility was required (e.g., Branch 1 Internet Backup).

---

## 1.3 Internet-Based Site-to-Site Backup

In the event of a total MPLS provider outage, the branches maintain connectivity via the public Internet:

- Branch 1: Leverages a GRE over IPsec tunnel.
- Branch 2: Leverages a VTI (Virtual Tunnel Interface) over IPsec for a more modern, interface-based VPN approach.

---

## 1.4 Intelligent Egress Redundancy (The "Transit" Solution)

For branches with only one physical Internet line, the MPLS link is used to borrow the Internet path of a neighbor:

- Tracking: IP SLA monitors local Internet reachability (pinging 8.8.8.8).
- Failover: If the local ISP fails, a floating static route redirects traffic over the MPLS backbone to a neighbor (CE2).
- PAT Redirection: The neighbor performs NAT/PAT for the transit traffic, ensuring the branch keeps Internet access even with a dead local ISP.

### Failover Mechanism (Branch 2/CE3 Example):

- Primary Path: CE3 reaches the internet directly via its local ISP (ISP_A).
- Health Monitoring: An IP SLA is configured to track reachability to 8.8.8.8.
- Static Route Tracking: The primary default route ($0.0.0.0/0$) is tied to the IP SLA state.
- Secondary Path (MPLS Transit): If the local ISP fails, the tracked route is pulled from the table. A floating static route (higher Administrative Distance) takes over, redirecting internet-bound traffic across the MPLS backbone to CE2.
- Transit NAT: CE2 receives the redirected traffic and performs PAT (Port Address Translation) to exit out of its own internet gateway, ensuring the branch remains online.

### Failover Mechanism (Branch 2/CE3 Example):

- Primary Path: CE3 reaches the internet directly via its local ISP (ISP_A).
- Health Monitoring: An IP SLA is configured to track reachability to 8.8.8.8.
- Static Route Tracking: The primary default route ($0.0.0.0/0$) is tied to the IP SLA state.
- Secondary Path (MPLS Transit): If the local ISP fails, the tracked route is pulled from the table. A floating static route (higher Administrative Distance) takes over, redirecting internet-bound traffic across the MPLS backbone to CE2.
- Transit NAT: CE2 receives the redirected traffic and performs PAT (Port Address Translation) to exit out of its own internet gateway, ensuring the branch remains online.

---

## 🏗️ Part 2: LAN Design & High Availability

The LAN follows Cisco’s Hierarchical Design Principles to ensure deterministic traffic flows and rapid convergence.

---
### 2.1 Multi-Tier Topology

- HQ Campus: Traditional 3-Tier Design (Access, Distribution, Core).
- HQ Data Center: 3-Tier Routed Access design to move the L3 boundary closer to servers, reducing the STP domain.
- Branch Offices: Collapsed Core design to optimize costs while maintaining performance.

---

### 2.2 Layer 2 Optimization & Management

- VTPv3: Provides rigid control via a Primary Server to synchronize the VLAN database and MST configurations.
- MST (Multiple Spanning Tree): Maps multiple VLANs into specific instances to load balance traffic across uplinks.
- DTP & EtherChannel: Dynamic trunking (DTP) and link aggregation (LACP/PAgP) are used for simplified deployment and bandwidth scaling.

---

### 2.3 Layer 3 Gateway & Conditional NAT

- FHRP (HSRP & VRRP): Deployed for gateway redundancy with tuned priorities for Active/Active load balancing.

#### NAT Exemption (ACL Logic):

- Deny: Traffic destined for internal private blocks (RFC 1918) is excluded from NAT to allow seamless VPN/MPLS routing.
- Permit: All other traffic is translated via PAT for public Internet access.
- Security: Refined ACLs ensure only authorized internal subnets can initiate NAT translations.
---

## 📊 Technical Stack Summary

| Layer | Technologies Used |
|------|-------------------|
| WAN | DMVPN Phase 3, IKEv2, IP SLA, Floating Static Routes, OSPF/EIGRP Redistribution |
| LAN L2 | MST, VTPv3 (Primary Server), DTP, LACP/PAgP |
| LAN L3 | HSRP, VRRP, NAT/PAT with ACL Exemption |
| Architecture | 3-Tier Campus, Routed Access (DC), Collapsed Core (Branch) |
