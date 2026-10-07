# ☕ Mike's Coffee Shop Network — Cisco Packet Tracer Lab

A small-business network build completed as part of my CCNA studies, covering VLAN segmentation, Layer 2 security, router-on-a-stick inter-VLAN routing, DHCP, extended ACLs, and secure remote management — configured entirely from scratch on Cisco IOS.

![Topology Diagram](topology/topology-diagram.png)

## 📋 Project Overview

This lab simulates the network for a small coffee shop ("Mike's Coffee Shop") with a single router and switch. Traffic is split into three purpose-built networks — office/management, point-of-sale, and free guest Wi-Fi — plus a dedicated network-management VLAN for the switch itself. The networks are deliberately isolated from one another to keep POS and staff systems safe from guest traffic.

## 🧰 Skills Demonstrated

- VLAN design and segmentation (4 VLANs across access + trunk ports)
- Switch port security (BPDU Guard + PortFast on all access ports)
- Router-on-a-Stick (dot1Q subinterfaces for inter-VLAN routing)
- SVI configuration for out-of-band switch management
- DHCP pool configuration with address exclusions for static infrastructure
- Extended, named Access Control Lists for guest network isolation
- SSH configuration and console/VTY line security
- Device hardening (encrypted passwords, MOTD banners, domain lookup disabled)

## 🗺️ Network Topology

| Device | Model | Role |
|---|---|---|
| MikesCoffee-R1 | Cisco 2911 | Gateway / Router-on-a-Stick / DHCP server |
| MikesCoffee-SW1 | Cisco Catalyst 3560-24PS | Access layer switch |
| AccessPoint-PT | — | Guest Wi-Fi |

**End devices:** Manager PC and office printer (VLAN 10), POS terminal and receipt printer (VLAN 20), two guest laptops over Wi-Fi (VLAN 30).

## 🔢 VLAN Design

| VLAN ID | Name | Purpose | Subnet |
|---|---|---|---|
| 10 | MANAGEMENT_OFFICE | Manager PC, office printer | 192.168.10.0/24 |
| 20 | POS | POS terminal, receipt printer | 192.168.20.0/24 |
| 30 | FREEWIFI | Guest Wi-Fi | 192.168.30.0/24 |
| 99 | NETWORK_MANAGEMENT | Switch management (SVI) | 192.168.99.0/24 |

Full breakdown, port assignments, and design rationale in [`docs/vlan-design.md`](./docs/vlan-design.md).

## 🔐 Security Configuration

- **Device access:** Console lines password-protected with `logging synchronous`; switch remote access is SSH-only (Telnet disabled) via a local `admin` account with privilege 15.
- **Port security:** PortFast + BPDU Guard on every access port to speed up convergence and shut down ports that unexpectedly receive BPDUs.
- **ACL:** `GUEST_RESTRICTIONS` on the router blocks guest VLAN (30) traffic from reaching the office, POS, or management VLANs, while still permitting DHCP and general internet access.

Full write-ups: [`docs/security-hardening.md`](./docs/security-hardening.md) · [`docs/acl-notes.md`](./docs/acl-notes.md)

## ⚙️ Configuration Files

Full `show running-config` output for each device is in [`/configs`](./configs) (line/enable passwords redacted for public posting; IOS-hashed secrets left as-is):
- [`switch1-config.txt`](./configs/switch1-config.txt)
- [`router1-config.txt`](./configs/router1-config.txt)

## 📁 Repository Structure

```
coffee-shop-network-lab/
├── topology/            # Network diagram
├── configs/              # Device running-configs
├── packet-tracer/        # .pkt lab file
└── docs/                  # VLAN design, security hardening, ACL notes
```

## 🧠 What I Learned

This was the first time I configured a switch and router entirely from scratch, rather than working from a pre-built lab — building up VLANs, trunking, SSH, and inter-VLAN routing step by step made concepts like 802.1Q encapsulation and router-on-a-stick click in a way that just reading about them hadn't. Writing the guest ACL also forced me to think carefully about rule order (the DHCP permit has to come before the deny statements, or guests would never get an address in the first place) and about *where* to apply an ACL for it to be both correct and efficient.

## 🚀 Next Steps

- [ ] Finish SSH configuration on the router (currently console/local only — see `security-hardening.md`)
- [ ] Add HSRP for gateway redundancy
- [ ] Add a second switch and test EtherChannel
- [ ] Move to per-technician local accounts instead of a shared `admin` login

---
*Built as part of my CCNA certification prep. Feedback welcome — feel free to open an issue or reach out.*
