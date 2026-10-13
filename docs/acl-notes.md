# Access Control Lists

## Purpose

The design goal is that the four networks cannot reach each other except where there is a deliberate reason. With router-on-a-stick, the router routes between every subinterface by default, so isolation does not exist until ACLs create it. Each VLAN gets its own extended, named ACL, applied **inbound** on that VLAN's subinterface.

| ACL | Applied to | What it enforces |
|---|---|---|
| `GUEST_RESTRICTIONS` | Gi0/1.30 in | Guests reach the internet only; no access to office, POS or management |
| `OFFICE_RESTRICTIONS` | Gi0/1.10 in | Office reaches the internet and the management VLAN; no access to POS or guest |
| `POS_RESTRICTIONS` | Gi0/1.20 in | POS reaches the internet only; no access to office, guest or management |
| `MGMT_RESTRICTIONS` | Gi0/1.99 in | Management VLAN can only answer the office and talk to its own gateway |

The resulting policy, read as "can the row start a connection to the column":

| From \ To | Office (10) | POS (20) | Guest (30) | Mgmt (99) | Internet |
|---|---|---|---|---|---|
| Office (10) | n/a | No | No | **Yes (deliberate)** | Yes |
| POS (20) | No | n/a | No | No | Yes |
| Guest (30) | No | No | n/a | No | Yes |
| Mgmt (99) | Replies only | No | No | n/a | No |

## Why inbound, on each VLAN's own subinterface

An inbound ACL filters packets as they enter the router from that VLAN, so it controls what the VLAN is allowed to *start*. Filtering close to the source means a blocked packet is dropped before it is routed anywhere, and each VLAN's policy lives in one obvious place. The alternative, outbound ACLs on the destination subinterfaces, would scatter one VLAN's rules across three interfaces.

## GUEST_RESTRICTIONS

```
ip access-list extended GUEST_RESTRICTIONS
 permit udp any eq bootpc any eq bootps
 deny ip 192.168.30.0 0.0.0.255 192.168.10.0 0.0.0.255
 deny ip 192.168.30.0 0.0.0.255 192.168.20.0 0.0.0.255
 deny ip 192.168.30.0 0.0.0.255 192.168.99.0 0.0.0.255
 permit ip 192.168.30.0 0.0.0.255 any
```

| # | Rule | Why |
|---|---|---|
| 1 | `permit udp any eq bootpc any eq bootps` | Lets DHCP through. See "Why every ACL needs a DHCP rule" below. |
| 2 | deny guest to 192.168.10.0/24 | Blocks the office network |
| 3 | deny guest to 192.168.20.0/24 | Blocks the POS network, the most sensitive segment |
| 4 | deny guest to 192.168.99.0/24 | Blocks management addresses |
| 5 | permit guest to any | Everything left is internet-bound traffic |

## OFFICE_RESTRICTIONS

```
ip access-list extended OFFICE_RESTRICTIONS
 permit udp any eq bootpc any eq bootps
 deny ip 192.168.10.0 0.0.0.255 192.168.20.0 0.0.0.255
 deny ip 192.168.10.0 0.0.0.255 192.168.30.0 0.0.0.255
 permit ip 192.168.10.0 0.0.0.255 any
```

There is deliberately **no deny for 192.168.99.0/24**. The Manager PC is where the network is administered from, and it has to reach the switch and router management addresses over SSH. This is a design decision with a cost: any device in VLAN 10 can attempt to connect to the management network, not only the Manager PC. See "Known limitations" below.

## POS_RESTRICTIONS

```
ip access-list extended POS_RESTRICTIONS
 permit udp any eq bootpc any eq bootps
 deny ip 192.168.20.0 0.0.0.255 192.168.10.0 0.0.0.255
 deny ip 192.168.20.0 0.0.0.255 192.168.30.0 0.0.0.255
 deny ip 192.168.20.0 0.0.0.255 192.168.99.0 0.0.0.255
 permit ip 192.168.20.0 0.0.0.255 any
```

The POS VLAN gets the same treatment as guest: internet only, nothing internal.

## MGMT_RESTRICTIONS

```
ip access-list extended MGMT_RESTRICTIONS
 permit tcp 192.168.99.0 0.0.0.255 192.168.10.0 0.0.0.255 established
 permit icmp 192.168.99.0 0.0.0.255 192.168.10.0 0.0.0.255 echo-reply
 permit ip 192.168.99.0 0.0.0.255 host 192.168.99.1
 deny ip 192.168.99.0 0.0.0.255 any
```

Almost nothing legitimate *starts* in the management VLAN, so this ACL is mostly about replies:

| # | Rule | Why |
|---|---|---|
| 1 | permit TCP to the office, `established` only | Lets the replies to an SSH session started from the office get back. `established` matches packets with the ACK or RST flag, which is every packet of a session except the first SYN, so the switch can answer but cannot open new connections. |
| 2 | permit ICMP echo-reply to the office | Lets the office ping the switch and get answers, but not the other way around |
| 3 | permit to the router's own address (192.168.99.1) | An inbound ACL also filters packets addressed to the router itself. Without this, the switch could not even ping its gateway. |
| 4 | explicit deny to anything else | The implicit deny would do the same. Writing it out gives it a hit counter in `show access-lists`, so blocked attempts are visible. |

There is no DHCP rule here because VLAN 99 uses static addressing only.

## Why every other ACL needs a DHCP rule

A device asking for an address does not have one yet, so its DHCP Discover is sent from `0.0.0.0` to `255.255.255.255` (UDP source port 68, destination port 67). That source address is not inside `192.168.30.0/24` or any other subnet-specific rule, so without an explicit permit the packet would fall through to the implicit deny at the end of the ACL, and no device on that VLAN could ever get an address. Rule 1 in each ACL fixes that.

## Known limitations and refinements

These are things I noticed on review. None break the lab, and they are listed so the trade-offs are visible.

1. **The DHCP permit is broader than it needs to be.** `permit udp any eq bootpc any eq bootps` matches *any* source and *any* destination with UDP ports 68 to 67. Because it sits above the deny rules, a packet crafted with those ports could slip past them, for example from the guest VLAN to a host in the POS VLAN. A tighter version matches only the Discover itself: `permit udp host 0.0.0.0 eq bootpc host 255.255.255.255 eq bootps`. Unicast renewals are already covered by the final `permit ip` rule.
2. **Devices can still reach the router's own address on their VLAN.** The guest ACL allows traffic to any destination outside the three denied subnets, and that includes 192.168.30.1, the router itself. So a guest can attempt an SSH login to the router. Authentication still protects it, but the better fix is an `access-class` on the VTY lines that restricts management sessions to the office VLAN. This is listed in the open items of [`security-hardening.md`](./security-hardening.md).
3. **The office VLAN can reach the entire management VLAN, not just the switch.** Locking this down to a single admin host needs a static address on the Manager PC.
4. **`established` is not real stateful inspection.** It looks at TCP flags on each packet independently, so a crafted packet with the ACK flag set would also match. Proper connection tracking needs reflexive ACLs or a zone-based firewall, both beyond this lab.

## Testing

The tests I ran and their results are in [`verification.md`](./verification.md).
