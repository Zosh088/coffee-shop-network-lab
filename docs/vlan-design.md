# VLAN Design

## Overview

The coffee shop's LAN is segmented into four VLANs on `MikesCoffee-SW1`, trunked to `MikesCoffee-R1` over `GigabitEthernet0/1` (802.1Q, VLANs 10/20/30/99 allowed). Each VLAN maps to its own subinterface on the router for inter-VLAN routing (router-on-a-stick), and each has its own DHCP pool except the management VLAN, which uses static addressing only.

## VLAN Table

| VLAN ID | Name | Purpose | Subnet | Gateway (Router Subinterface) |
|---|---|---|---|---|
| 10 | MANAGEMENT_OFFICE | Manager PC, office printer | 192.168.10.0/24 | Gi0/1.10 — 192.168.10.1 |
| 20 | POS | Point-of-sale terminal, receipt printer | 192.168.20.0/24 | Gi0/1.20 — 192.168.20.1 |
| 30 | FREEWIFI | Guest Wi-Fi (via access point) | 192.168.30.0/24 | Gi0/1.30 — 192.168.30.1 |
| 99 | NETWORK_MANAGEMENT | Switch management (SVI) | 192.168.99.0/24 | Gi0/1.99 — 192.168.99.1 |

## Switch Port Assignments

| Interface(s) | VLAN | Connected Device(s) |
|---|---|---|
| Fa0/1 – Fa0/5 | 10 (MANAGEMENT_OFFICE) | Manager PC, office printer, spare office ports |
| Fa0/6 – Fa0/10 | 20 (POS) | POS terminal, receipt printer, spare POS ports |
| Fa0/11 | 30 (FREEWIFI) | Guest Wi-Fi access point |
| Gi0/1 | Trunk (10, 20, 30, 99) | Uplink to MikesCoffee-R1 |
| Vlan99 (SVI) | 99 | Switch management IP: 192.168.99.2/24 |

## Why This Design

- **Traffic separation by function, not just device type.** POS traffic (card transactions) stays isolated from general office traffic and guest Wi-Fi — reduces both accidental interference and attack surface if a guest device is compromised.
- **VLAN 99 for management** keeps switch administration off the same broadcast domain as any user-facing VLAN, following the common "don't manage infrastructure from VLAN 1 or a user VLAN" practice.
- **DHCP scoped per VLAN** on the router, with the first 20 addresses in each /24 excluded and reserved for static-IP infrastructure (printers, etc.), avoiding address conflicts.
- **Router-on-a-stick** was used instead of SVIs on the switch for inter-VLAN routing because the switch in this lab (3560-24PS) was configured primarily as a Layer 2 access/distribution device with one Layer 3 management interface (Vlan99) — routing between VLANs is centralized on `MikesCoffee-R1`.

## IP Addressing Notes

- **Office Printer:** static `192.168.10.10/24`, gateway `192.168.10.1`, DNS `8.8.8.8`
- **Receipt Printer:** static `192.168.20.10/24`, gateway `192.168.20.1`, DNS `8.8.8.8`
- All other devices in VLANs 10, 20, and 30 receive addresses dynamically from the router's DHCP pools.
