# VLAN Design

## Overview

The coffee shop LAN is segmented into four working VLANs on `MikesCoffee-SW1`, plus two "housekeeping" VLANs that exist only for security reasons. The switch connects to `MikesCoffee-R1` over a single 802.1Q trunk (`GigabitEthernet0/1`). Each working VLAN has its own subinterface on the router (router-on-a-stick), which acts as that VLAN's default gateway and as the single point where traffic between VLANs is routed and filtered.

## VLAN Table

| VLAN ID | Name | Purpose | Subnet | Gateway (router subinterface) |
|---|---|---|---|---|
| 10 | MANAGEMENT | Manager PC, office printer | 192.168.10.0/24 | Gi0/1.10, 192.168.10.1 |
| 20 | POS | POS terminal, receipt printer | 192.168.20.0/24 | Gi0/1.20, 192.168.20.1 |
| 30 | FREE_WIFI | Guest Wi-Fi (via access point) | 192.168.30.0/24 | Gi0/1.30, 192.168.30.1 |
| 99 | NETWORK_MANAGEMENT | Switch management interface (SVI) and the technician maintenance port | 192.168.99.0/24 | Gi0/1.99, 192.168.99.1 |
| 998 | NATIVE_UNUSED | Native VLAN on the trunk; carries nothing | none | none |
| 999 | PARKING_LOT | Holding VLAN for disabled, unassigned ports | none | none |

VLANs 998 and 999 have no subnet and no gateway on purpose. Nothing can be routed in or out of them.

> The router's DHCP pools use their own labels (`MANAGEMENT_OFFICE`, `POS`, `FREEWIFI`). Those are pool names on the router and are independent of the VLAN names on the switch.

## Switch Port Assignments

| Interface(s) | VLAN | State | Connected device(s) |
|---|---|---|---|
| Fa0/1 - Fa0/2 | 10 | Up | Manager PC, office printer (port security, sticky MAC) |
| Fa0/3 - Fa0/5 | 10 | Shut down | Reserved spares |
| Fa0/6 - Fa0/7 | 20 | Up | POS terminal, receipt printer (port security, sticky MAC) |
| Fa0/8 - Fa0/10 | 20 | Shut down | Reserved spares |
| Fa0/11 | 30 | Up | Guest Wi-Fi access point (no port security: it bridges many clients) |
| Fa0/15 | 99 | Shut down (enabled only during maintenance) | Technician laptop, 192.168.99.10 (port security bound to its MAC) |
| Fa0/12 - Fa0/14 and Fa0/16 - Fa0/24 | 999 | Shut down | Unassigned |
| Gi0/1 | Trunk | Up | Uplink to MikesCoffee-R1 (native VLAN 998, allowed 10,20,30,99) |
| Gi0/2 | 999 | Shut down | Unassigned |
| Vlan99 (SVI) | 99 | Up | Switch management address 192.168.99.2/24 |

Guest laptops connect wirelessly through the access point and have no switch ports of their own.

### Why the spare ports are handled in two different ways

- **Reserved spares (Fa0/3-5, Fa0/8-10)** are shut down but stay in their own VLAN. They are meant for devices the shop will plausibly add later. A `no shutdown` brings one back into the right network with no other changes.
- **Unassigned ports (Fa0/12-14, Fa0/16-24, Gi0/2)** have no planned purpose, so they are shut down and moved to VLAN 999. If someone re-enables one by mistake, the device on it lands in a VLAN with no gateway, no DHCP and no neighbours.

## Addressing

- **DHCP** is served by the router, one pool per user VLAN. The first 20 addresses in each /24 are excluded from the pools and reserved for devices that need static addresses.
- **Office printer:** static 192.168.10.10/24, gateway 192.168.10.1, DNS 8.8.8.8
- **Receipt printer:** static 192.168.20.10/24, gateway 192.168.20.1, DNS 8.8.8.8
- **VLAN 99** has no DHCP pool. Everything in it is static: router subinterface 192.168.99.1, switch SVI 192.168.99.2, technician laptop 192.168.99.10 (gateway 192.168.99.1).

## Design Decisions

- **Separation by function.** POS traffic stays away from guest devices and general office traffic. If a guest device is compromised, it has no direct path to the payment systems.
- **A dedicated management VLAN.** Administration of the switch happens in VLAN 99, so no user-facing VLAN shares a broadcast domain with the infrastructure's management addresses.
- **A maintenance port instead of an admin PC on the office network.** Administration is done from a technician laptop plugged into Fa0/15 in VLAN 99, and that port is shut down except during maintenance. The manager's daily-use PC has no administrative access. Each of VLANs 10 and 20 also keeps three reserved spare ports.
- **Router-on-a-stick.** A Catalyst 3560 is a multilayer switch and could route between VLANs itself using SVIs and `ip routing`, which would avoid sending all inter-VLAN traffic over one trunk. I used router-on-a-stick because it is a core CCNA topic and I wanted to build it by hand. It also keeps routing and ACLs on one device, which keeps the security policy easy to read. At a coffee shop's traffic levels, a single gigabit trunk is not a bottleneck.
- **Unused VLANs kept off the trunk.** The trunk's allowed list contains only `10,20,30,99`. VLANs 998 and 999 never cross it.

## Router Interfaces

| Interface | Role | State |
|---|---|---|
| Gi0/0 | Uplink to ISP (`## TO_ISP ##`) | No IP address yet; waiting for the ISP side to be simulated |
| Gi0/1 | Trunk to the switch; parent of the four subinterfaces | Up, no IP address of its own |
| Gi0/1.10 / .20 / .30 / .99 | Gateways for VLANs 10 / 20 / 30 / 99 | Up, each with an inbound ACL |
| Gi0/2 | Unused | Shut down |

The VLAN definitions themselves are stored in the switch's VLAN database, not in its running-config, so they do not appear in [`configs/switch1-config.txt`](../configs/switch1-config.txt). Use `show vlan brief` on the switch to see them.
