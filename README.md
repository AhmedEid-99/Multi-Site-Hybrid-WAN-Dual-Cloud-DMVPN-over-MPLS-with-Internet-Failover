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

- Conditional NAT/PAT: Configured at the WAN edge to translate internal private addressing to public-routable IP addresses.

- NAT Exemption (ACL Logic): A refined Access Control List is used to "deny" NAT for any traffic destined for internal private network blocks (RFC 1918). This prevents the router from translating IP addresses when sending data over the DMVPN or MPLS tunnels.

- Internet Translation: The ACL "permits" all other traffic, allowing standard PAT for any traffic destined for the public Internet.

- Security: Refined ACLs ensure that only authorized internal subnets can initiate a NAT translation, reducing th
