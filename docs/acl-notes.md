# Access Control List — GUEST_RESTRICTIONS

## Purpose

Guests connecting via the `FREEWIFI` VLAN (30) should be able to reach the internet, but must **not** be able to reach any internal VLAN — management office (10), POS (20), or network management (99). This keeps customer devices from ever touching staff systems or payment infrastructure.

## Configuration

Applied as an extended, named ACL on the router, inbound on the guest gateway subinterface:

```
ip access-list extended GUEST_RESTRICTIONS
 permit udp any eq bootpc any eq bootps
 deny ip 192.168.30.0 0.0.0.255 192.168.10.0 0.0.0.255
 deny ip 192.168.30.0 0.0.0.255 192.168.20.0 0.0.0.255
 deny ip 192.168.30.0 0.0.0.255 192.168.99.0 0.0.0.255
 permit ip 192.168.30.0 0.0.0.255 any
!
interface GigabitEthernet0/1.30
 ip access-group GUEST_RESTRICTIONS in
```

## Rule-by-Rule Breakdown

| # | Rule | Why |
|---|---|---|
| 1 | `permit udp any eq bootpc any eq bootps` | Allows DHCP request/reply traffic (BOOTP client port 68 → server port 67) through *before* the deny rules, so guest devices can still get an IP address via DHCP relay/broadcast. Without this, DHCP could be blocked by the broader deny statements below. |
| 2 | `deny ip 192.168.30.0/24 → 192.168.10.0/24` | Blocks guest traffic from reaching the management/office network. |
| 3 | `deny ip 192.168.30.0/24 → 192.168.20.0/24` | Blocks guest traffic from reaching the POS network — the most security-sensitive segment. |
| 4 | `deny ip 192.168.30.0/24 → 192.168.99.0/24` | Blocks guest traffic from reaching switch/router management IPs. |
| 5 | `permit ip 192.168.30.0/24 → any` | Explicitly allows everything else (i.e., internet-bound traffic) once the three internal networks are excluded. |

## Placement Logic

The ACL is applied **inbound** on the `Gi0/1.30` subinterface (the guest gateway) rather than outbound on the other VLANs' subinterfaces. Filtering as close to the traffic source as possible is generally more efficient and easier to reason about — one ACL on one interface covers all guest-originated traffic, rather than needing matching deny rules replicated across the management, POS, and network-management gateways.

## Verification Ideas (for future testing)

- From a guest laptop, `ping 192.168.10.1` and `192.168.20.1` — should fail.
- From a guest laptop, confirm DHCP-assigned addressing works and internet-bound traffic (e.g. `ping 8.8.8.8`) succeeds.
- Try `telnet 192.168.99.2` from a guest device — should be blocked by the ACL (and would be refused anyway, since only SSH is accepted).
