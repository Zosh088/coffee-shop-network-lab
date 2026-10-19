# Access Control Lists

## Purpose

The design goal is that the four networks cannot reach each other at all, and that only one machine can administer the infrastructure. With router-on-a-stick, the router routes between every subinterface by default, so isolation does not exist until ACLs create it. Each VLAN gets its own extended, named ACL, applied **inbound** on that VLAN's subinterface. A separate standard ACL protects the devices' own management logins.

| ACL | Applied to | What it enforces |
|---|---|---|
| `GUEST_RESTRICTIONS` | Gi0/1.30 in | Guests reach the internet only; no access to office, POS or management |
| `OFFICE_RESTRICTIONS` | Gi0/1.10 in | Office reaches the internet only; no access to POS, guest or management |
| `POS_RESTRICTIONS` | Gi0/1.20 in | POS reaches the internet only; no access to office, guest or management |
| `MGMT_RESTRICTIONS` | Gi0/1.99 in | The management VLAN can only reach its own gateway address |
| `VTY_ADMIN` (standard) | VTY lines on both devices, via `access-class` | Only the technician laptop may open a management session to the router or switch |

The resulting policy, read as "can the row start a connection to the column":

| From \ To | Office (10) | POS (20) | Guest (30) | Mgmt (99) | Internet |
|---|---|---|---|---|---|
| Office (10) | n/a | No | No | No | Yes |
| POS (20) | No | n/a | No | No | Yes |
| Guest (30) | No | No | n/a | No | Yes |
| Mgmt (99) | No | No | No | n/a | No |

The internet column is the policy. The ISP side is not built yet (see the open items in [`security-hardening.md`](./security-hardening.md)).

## Why inbound, on each VLAN's own subinterface

An inbound ACL filters packets as they enter the router from that VLAN, so it controls what the VLAN is allowed to *start*. Filtering close to the source means a blocked packet is dropped before it is routed anywhere, and each VLAN's policy lives in one obvious place. The alternative, outbound ACLs on the destination subinterfaces, would scatter one VLAN's rules across three interfaces.

## The three user-VLAN ACLs

Guest, office and POS share one shape: a DHCP permit, a deny for every other internal subnet, and a permit for everything else (the internet).

```
ip access-list extended GUEST_RESTRICTIONS
 permit udp host 0.0.0.0 eq bootpc host 255.255.255.255 eq bootps
 deny ip 192.168.30.0 0.0.0.255 192.168.10.0 0.0.0.255
 deny ip 192.168.30.0 0.0.0.255 192.168.20.0 0.0.0.255
 deny ip 192.168.30.0 0.0.0.255 192.168.99.0 0.0.0.255
 permit ip 192.168.30.0 0.0.0.255 any

ip access-list extended OFFICE_RESTRICTIONS
 permit udp host 0.0.0.0 eq bootpc host 255.255.255.255 eq bootps
 deny ip 192.168.10.0 0.0.0.255 192.168.20.0 0.0.0.255
 deny ip 192.168.10.0 0.0.0.255 192.168.30.0 0.0.0.255
 deny ip 192.168.10.0 0.0.0.255 192.168.99.0 0.0.0.255
 permit ip 192.168.10.0 0.0.0.255 any

ip access-list extended POS_RESTRICTIONS
 permit udp host 0.0.0.0 eq bootpc host 255.255.255.255 eq bootps
 deny ip 192.168.20.0 0.0.0.255 192.168.10.0 0.0.0.255
 deny ip 192.168.20.0 0.0.0.255 192.168.30.0 0.0.0.255
 deny ip 192.168.20.0 0.0.0.255 192.168.99.0 0.0.0.255
 permit ip 192.168.20.0 0.0.0.255 any
```

| Rule | Why |
|---|---|
| DHCP permit | Lets a device with no address yet get one. See the DHCP section below. |
| Deny to each other internal subnet | Blocks the other user networks and the management network. POS is the most sensitive segment, so no other VLAN can reach it. |
| Permit to any | Everything left is internet-bound traffic. |

The office VLAN has **no exception** for the management network. An earlier version allowed it so the Manager PC could administer the devices. That was removed once a dedicated technician laptop took over administration, so no office device, including the manager's daily-use PC, can reach management addresses.

## MGMT_RESTRICTIONS

```
ip access-list extended MGMT_RESTRICTIONS
 permit ip 192.168.99.0 0.0.0.255 host 192.168.99.1
 deny ip 192.168.99.0 0.0.0.255 any
```

| Rule | Why |
|---|---|
| Permit to the router's own address (192.168.99.1) | An inbound ACL also filters packets addressed to the router itself. Without this, the technician laptop could not SSH to the router, and the switch could not ping its gateway. |
| Explicit deny to anything else | The implicit deny would do the same. Writing it out gives it a hit counter in `show access-lists`, so blocked attempts are visible. |

No reply rules are needed. The technician laptop and the switch are in the same VLAN, so they talk at Layer 2 and the router never sees that traffic. The router's own replies (for example the SSH session to the router) originate on the router, and ACLs do not filter packets the router itself generates. Nothing in VLAN 99 needs to be answered across a VLAN boundary.

## VTY_ADMIN

```
ip access-list standard VTY_ADMIN
 permit host 192.168.99.10
 deny any
!
line vty 0 15
 access-class VTY_ADMIN in
```

The four ACLs above control traffic passing *through* the router. They do not control who can open a management session *to* the router or switch. `VTY_ADMIN` does that, and only the technician laptop (192.168.99.10) may start a session.

A standard ACL is enough because only the source of the connection matters. It is applied with `access-class` on the VTY lines, which is different from `ip access-group` on an interface. It is configured on both devices. On the switch it is defence in depth, since `MGMT_RESTRICTIONS` and the other ACLs already keep every routed VLAN away from VLAN 99.

## Why the user-VLAN ACLs need a DHCP rule, and why it is so narrow

A device asking for an address does not have one yet, so its DHCP Discover is sent from `0.0.0.0` to `255.255.255.255` (UDP source port 68, destination port 67). That source address is not inside any subnet-specific rule, so without an explicit permit the packet would fall through to the implicit deny at the end of the ACL, and no device on that VLAN could ever get an address.

The first version of this rule was `permit udp any eq bootpc any eq bootps`, which matched *any* source and *any* destination with those ports. Because it sat above the deny rules, a crafted packet with UDP ports 68 to 67 could have slipped past them. The rule now matches only the Discover itself: source `0.0.0.0`, destination `255.255.255.255`. Renewals from a device that already has an address are ordinary traffic and are covered by each ACL's final `permit ip` rule.

Packet Tracer cannot craft a custom UDP packet, so the old bypass could not be demonstrated and its closure cannot be either. What was verified is that DHCP still works on all three user VLANs with the tighter rule (see [`verification.md`](./verification.md)).

## Known limitations

1. **Administrator access is checked by IP address.** A device plugged into a VLAN 99 port and set to 192.168.99.10 would pass `VTY_ADMIN`. The maintenance port is shut down except during maintenance, which is the main mitigation. Port security, planned next, will also bind that port to the laptop's MAC address.
2. **Devices can still reach the router's own address on their VLAN.** The guest ACL allows traffic to any destination outside the three denied subnets, and that includes 192.168.30.1, the router itself, so a guest can still ping it. SSH sessions from anywhere except the technician laptop are refused by `VTY_ADMIN`.
3. **The internet-facing side has no ACL yet.** When the ISP interface is configured, it will need its own inbound ACL to block spoofed and unsolicited traffic.

## Testing

The tests I ran and their results are in [`verification.md`](./verification.md), and the command output is in [`evidence.md`](./evidence.md).
