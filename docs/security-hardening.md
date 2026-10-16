# Security Hardening

This lab was built in two passes. The first pass followed a tutorial and produced a working, segmented network. The second pass was an audit of my own configs, where I read both running-configs line by line, compared them to the design goals, and fixed the gaps I found. This document covers both.

## Hardening pass: findings and fixes

| # | Finding | Why it mattered | Fix |
|---|---|---|---|
| 1 | The router had no working remote access. Its VTY lines had a bare `login` and nothing else. | It could not be managed remotely, and it did not match the switch. | SSHv2 only, local `admin` account, 2048-bit RSA key, Telnet refused |
| 2 | Only the guest VLAN had an ACL. | Router-on-a-stick routes between all VLANs by default, so office, POS and management could all reach each other, contradicting the design. | One inbound ACL per VLAN. See [`acl-notes.md`](./acl-notes.md). |
| 3 | RSA key size was never specified. | Without a modulus size, IOS can generate a short, weak key. | Regenerated at 2048 bits on both devices |
| 4 | Idle-session timeouts relied on defaults. | An admin session left open on an unattended PC stays valid for the default 10 minutes. | Explicit `exec-timeout 5 0` on the switch's console and VTY lines |
| 5 | Unused ports were live in VLAN 1. | Anyone could plug in and get a working port. | Unassigned ports shut down and moved to VLAN 999; reserved spares shut down in their own VLANs |
| 6 | The trunk's native VLAN was the default, VLAN 1, and the trunk still sent DTP advertisements. | The native VLAN is the basis of double-tagging VLAN hopping, and DTP lets a connected device try to negotiate a trunk. | Native VLAN set to the unused VLAN 998; `switchport nonegotiate` on the trunk |
| 7 | Any device that could reach the router or switch could try to open an SSH session, including guest and POS devices connecting to the router's address on their own VLAN. | The inter-VLAN ACLs filter traffic passing *through* the router, not sessions opened *to* it, so authentication was the only barrier. | Standard ACL `VTY_ADMIN` applied with `access-class` on the VTY lines of both devices, so only the office VLAN may open a management session |
| 8 | The router's console and VTY lines still used the default 10-minute timeout while the switch used 5. | Inconsistent policy between the two devices. | Explicit `exec-timeout 5 0` on the router's console and VTY lines |

One lesson from this pass: IOS does not print settings that are at their default values. An absent line in the running-config does not mean a setting is missing, and I had to check with the specific `show` command or `show running-config all` before calling something a gap.

## Device access

| | MikesCoffee-SW1 | MikesCoffee-R1 |
|---|---|---|
| Console | Password set, `login`, `logging synchronous`, 5-minute timeout | Password set, `login`, `logging synchronous`, 5-minute timeout |
| Privileged EXEC | `enable secret` (hashed) | `enable secret` (hashed) |
| Remote access | SSHv2 only, local `admin` account | SSHv2 only, local `admin` account |
| VTY lines | 0-15: `login local`, `transport input ssh`, 5-minute timeout | 0-15: `login local`, `transport input ssh`, 5-minute timeout |
| Who may open a session | Office VLAN only (`access-class VTY_ADMIN`) | Office VLAN only (`access-class VTY_ADMIN`) |
| Banner | MOTD warning | MOTD warning |

- **`service password-encryption`** is on for both devices. It stores line passwords as Type 7, which is reversible and only keeps them from being read at a glance. `enable secret` and `username ... secret` use a one-way hash, and should be used for anything that matters. This is why the console password in the running-config is weaker than the others.
- **`no ip domain-lookup`** stops the CLI from treating mistyped commands as hostnames to resolve, which otherwise causes a long delay after every typo.
- **SSH requires four things on IOS:** a hostname, a domain name, an RSA key pair and a local user. The domain name is needed because IOS names the key `hostname.domain`.
- **`ip ssh version 2`** disables SSHv1, which has known cryptographic weaknesses.
- **`transport input ssh`** refuses Telnet, so credentials never cross the network in cleartext.

## Layer 2 protections

- **PortFast** on every access port that connects to an end device. These ports skip the spanning-tree listening and learning delay and come up immediately.
- **BPDU Guard** on the same ports. A port set up for an end device should never receive a spanning-tree BPDU. If one does, someone has connected a switch, and BPDU Guard puts the port into `err-disabled` instead of letting a stranger influence the topology.
- **Unused ports** are shut down and, where they have no purpose, moved to VLAN 999 with `switchport mode access`. Three layers cover for each other: `shutdown` turns the port off; VLAN 999 is a dead end if it is re-enabled; and hard-setting access mode stops the port negotiating a trunk with whatever is plugged in.
- **Trunk hardening:** the trunk to the router is `mode trunk` with `nonegotiate`, an allowed list of only `10,20,30,99`, and native VLAN 998. Nothing legitimate is untagged on this trunk, so a stray untagged frame has nowhere useful to go.

> **A terminology note.** PortFast and BPDU Guard are spanning-tree features. Cisco's feature called "port security" is something different (`switchport port-security`, which limits and tracks the MAC addresses on a port). I have not implemented it yet; it is in the open items.

## Layer 3 protections

Inter-VLAN filtering is covered in [`acl-notes.md`](./acl-notes.md).

## Management access

- The switch is managed through an SVI in VLAN 99 (`192.168.99.2`). This is **in-band** management, because it uses the same network as user traffic. The console port is the out-of-band path.
- The Manager PC in VLAN 10 is permitted through the ACLs to reach VLAN 99. That is the only intended administration path.
- **`access-class` is different from `ip access-group`.** `ip access-group` filters traffic *passing through* an interface. `access-class` on the VTY lines filters who may open a management session *to the device itself*. Both are needed: the interface ACLs stop guests reaching the management VLAN, and `VTY_ADMIN` stops them logging in to the router through its own address on their VLAN.
- On the switch, `VTY_ADMIN` is defence in depth, because the guest and POS ACLs already block those VLANs from reaching VLAN 99. If one control is changed or fails, the other still holds.

## Open items

These are known gaps. They are listed rather than hidden.

1. **Unique credentials per device.** The `enable secret` and console password are the same on both devices, which came from the tutorial. A password reused across devices means one compromise opens both.
2. **Per-technician accounts** instead of a shared `admin` login, for accountability.
3. **Narrow admin access to a single host.** `VTY_ADMIN`, and the office ACL's access to VLAN 99, both trust the whole 192.168.10.0/24 network, not only the Manager PC. Tightening this needs a static address on the Manager PC.
4. **Implement Cisco port security** (`switchport port-security`, sticky MAC) on the access ports.
5. **Tighten the DHCP rule** in each ACL so it matches only the DHCP Discover (details in `acl-notes.md`).
6. **Add DHCP snooping and Dynamic ARP Inspection** to protect against rogue DHCP servers and ARP spoofing.
7. **Finish the ISP side:** a default route and NAT on `Gi0/0`.

## A note on the credentials in this repo

All passwords in this lab are throwaway values for a simulation. The running-configs in [`/configs`](../configs) have every secret, hash and line password redacted. The `.pkt` file itself stores the saved device configurations, so anyone who opens it can see them. That is acceptable only because these are lab credentials. They should never be reused on real equipment.
