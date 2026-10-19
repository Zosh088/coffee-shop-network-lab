# Security Hardening

This lab was built in two passes. The first pass followed a tutorial and produced a working, segmented network. The second pass was an audit of my own configs, where I read both running-configs line by line, compared them to the design goals, and fixed the gaps I found. This document covers both.

## Hardening pass: findings and fixes

| # | Finding | Why it mattered | Fix |
|---|---|---|---|
| 1 | The router had no working remote access. Its VTY lines had a bare `login` and nothing else. | It could not be managed remotely, and it did not match the switch. | SSHv2 only, local accounts, 2048-bit RSA key, Telnet refused |
| 2 | Only the guest VLAN had an ACL. | Router-on-a-stick routes between all VLANs by default, so office, POS and management could all reach each other, contradicting the design. | One inbound ACL per VLAN. See [`acl-notes.md`](./acl-notes.md). |
| 3 | RSA key size was never specified. | Without a modulus size, IOS can generate a short, weak key. | Regenerated at 2048 bits on both devices |
| 4 | Idle-session timeouts relied on the 10-minute default, and the router and switch were not configured alike. | An admin session left open on an unattended PC stays valid for the full default. | Explicit `exec-timeout 5 0` on the console and VTY lines of both devices |
| 5 | Unused ports were live in VLAN 1. | Anyone could plug in and get a working port. | Unassigned ports shut down and moved to VLAN 999; reserved spares shut down in their own VLANs |
| 6 | The trunk's native VLAN was the default, VLAN 1, and the trunk still sent DTP advertisements. | The native VLAN is the basis of double-tagging VLAN hopping, and DTP lets a connected device try to negotiate a trunk. | Native VLAN set to the unused VLAN 998; `switchport nonegotiate` on the trunk |
| 7 | Any device that could reach the router or switch could try to open an SSH session, including guest and POS devices connecting to the router's address on their own VLAN. | The inter-VLAN ACLs filter traffic passing *through* the router, not sessions opened *to* it, so authentication was the only barrier. | Standard ACL `VTY_ADMIN` applied with `access-class` on the VTY lines of both devices |
| 8 | Both devices shared one `admin` account. | A login could not be traced to a person, and one leaked password opened everything. | Named accounts `tech1` and `tech2`, each with its own password on each device; `admin` removed |
| 9 | The `enable secret` and the console password were identical on both devices (the hashes matched). The console password was also stored in reversible Type 7. | One compromised password opened both devices. | A unique `enable secret` per device (the hashes now differ); the console uses `login local`, so the shared console password no longer exists |
| 10 | Administration was allowed from the whole office VLAN, including the manager's daily-use PC. | The most valuable credentials were typed on the machine most exposed to email and browsing, and any office device could attempt a login. | A dedicated technician laptop on a maintenance port in the management VLAN; `VTY_ADMIN` permits only that host; the office-to-management exception removed |
| 11 | The DHCP permit in each user-VLAN ACL matched any source and any destination. | A crafted packet with UDP ports 68 to 67 could slip past the deny rules beneath it. | Rule narrowed to the DHCP Discover itself, from `0.0.0.0` to `255.255.255.255` |

Two lessons from this pass:

- IOS does not print settings that are at their default values. An absent line in the running-config does not mean a setting is missing, and I had to check with the specific `show` command or `show running-config all` before calling something a gap.
- A mistyped ACL name does not produce an error. IOS creates a new, empty ACL under the misspelled name. I found a stray one while reviewing `show access-lists`, and removed it.

## Device access

| | MikesCoffee-SW1 | MikesCoffee-R1 |
|---|---|---|
| Console | `login local`, `logging synchronous`, 5-minute timeout | `login local`, `logging synchronous`, 5-minute timeout |
| Privileged EXEC | `enable secret` (hashed, unique to this device) | `enable secret` (hashed, unique to this device) |
| Accounts | `tech1`, `tech2` (privilege 15), passwords unique to this device | `tech1`, `tech2` (privilege 15), passwords unique to this device |
| Remote access | SSHv2 only | SSHv2 only |
| VTY lines | 0-15: `login local`, `transport input ssh`, 5-minute timeout | 0-15: `login local`, `transport input ssh`, 5-minute timeout |
| Who may open a session | Technician laptop only (`access-class VTY_ADMIN`) | Technician laptop only (`access-class VTY_ADMIN`) |
| Banner | MOTD warning | MOTD warning |

- **`secret` stores a one-way hash.** These devices produce Type 5 (MD5-based) hashes. Newer IOS releases support stronger types, which I would use on real hardware.
- **`service password-encryption`** is on for both devices. With the console password gone, the configs no longer contain any Type 7 passwords, but it stays on so that anything added later is not stored in cleartext. Type 7 is reversible, so it is not real protection.
- **`no ip domain-lookup`** stops the CLI from treating mistyped commands as hostnames to resolve, which otherwise causes a long delay after every typo.
- **SSH requires four things on IOS:** a hostname, a domain name, an RSA key pair and a local user. The domain name is needed because IOS names the key `hostname.domain`.
- **`ip ssh version 2`** disables SSHv1, which has known cryptographic weaknesses.
- **`transport input ssh`** refuses Telnet, so credentials never cross the network in cleartext.
- **Console `login local`** means the console asks for a username and password, like SSH. The console is the out-of-band path, so it is the one that still works when the network or an ACL is misconfigured.

## Layer 2 protections

- **PortFast** on every access port that connects to an end device. These ports skip the spanning-tree listening and learning delay and come up immediately.
- **BPDU Guard** on the same ports. A port set up for an end device should never receive a spanning-tree BPDU. If one does, someone has connected a switch, and BPDU Guard puts the port into `err-disabled` instead of letting a stranger influence the topology.
- **Unused ports** are shut down and, where they have no purpose, moved to VLAN 999 with `switchport mode access`. Three layers cover for each other: `shutdown` turns the port off; VLAN 999 is a dead end if it is re-enabled; and hard-setting access mode stops the port negotiating a trunk with whatever is plugged in.
- **The maintenance port (Fa0/15)** is an access port in VLAN 99 with PortFast and BPDU Guard, and it is shut down except during maintenance.
- **Trunk hardening:** the trunk to the router is `mode trunk` with `nonegotiate`, an allowed list of only `10,20,30,99`, and native VLAN 998. Nothing legitimate is untagged on this trunk, so a stray untagged frame has nowhere useful to go.

> **A terminology note.** PortFast and BPDU Guard are spanning-tree features. Cisco's feature called "port security" is something different (`switchport port-security`, which limits and tracks the MAC addresses on a port). I have not implemented it yet; it is in the open items.

## Layer 3 protections

Inter-VLAN filtering is covered in [`acl-notes.md`](./acl-notes.md).

## Management access

- The switch is managed through an SVI in VLAN 99 (`192.168.99.2`). This is **in-band** management, because it uses the same network as user traffic. The console port is the out-of-band path.
- **Only the technician laptop can administer the network.** It sits on a maintenance port (Fa0/15) in VLAN 99 with the static address 192.168.99.10. The manager's PC has no administrative access of any kind.
- **Why a dedicated machine.** The manager's PC is a daily-use machine with email and browsing, and it is the last place administrator credentials should be typed. In a real shop, the equivalent is an IT provider's laptop that is plugged in only for maintenance, or remote management over a VPN.
- **`access-class` is different from `ip access-group`.** `ip access-group` filters traffic *passing through* an interface. `access-class` on the VTY lines filters who may open a management session *to the device itself*. Both are needed: the interface ACLs keep every routed VLAN away from the management VLAN, and `VTY_ADMIN` stops anything else logging in to the router through its own address on a user VLAN.
- On the switch, `VTY_ADMIN` is defence in depth, because the interface ACLs already keep routed VLANs away from VLAN 99. If one control is changed or fails, the other still holds.

### Maintenance procedure

1. At the switch console, enable the maintenance port: `interface fastEthernet 0/15`, then `no shutdown`.
2. From the technician laptop, SSH to the router or switch with a personal account (`tech1` or `tech2`).
3. When finished, shut the port again with `shutdown`.

The port is shut by default, so the first barrier is physical presence at the switch. The source-address check and the named account come after it.

## Open items

These are known gaps. They are listed rather than hidden.

1. **Cisco port security** (`switchport port-security`, sticky MAC) on the access ports, including binding the maintenance port to the technician laptop's MAC address.
2. **DHCP snooping and Dynamic ARP Inspection** to protect against rogue DHCP servers and ARP spoofing. The two printers have static addresses, so DAI would need ARP ACLs for them.
3. **Finish the ISP side:** a default route and NAT on `Gi0/0`, with an inbound ACL on that interface.
4. **Redundancy:** a second switch with EtherChannel, and HSRP for the gateway. This changes the topology, so it will be a separate lab.
5. **Centralised authentication** (AAA with RADIUS or TACACS+) instead of local accounts on each device. This is a better answer than maintaining separate passwords everywhere, and it is what real networks use.

## A note on the credentials in this repo

All passwords in this lab are throwaway values for a simulation. The running-configs in [`/configs`](../configs) have every secret and hash redacted. The `.pkt` file itself stores the saved device configurations, so anyone who opens it can see them. That is acceptable only because these are lab credentials. They should never be reused on real equipment.
