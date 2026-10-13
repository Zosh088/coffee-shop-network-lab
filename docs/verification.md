# Verification

How I tested each change. Every hardening step was tested before the configuration was saved and committed, so a failure could be traced to the change that caused it. Packet Tracer often times out on the first ping while ARP resolves, so each ping was repeated.

## Router SSH

| Test | Expected | Result |
|---|---|---|
| Manager PC: `ssh -l admin 192.168.99.1` | Login prompt, then router CLI | Pass |

## Inter-VLAN ACLs: office and POS

| From | Test | Expected | Result |
|---|---|---|---|
| Manager PC | `ping 192.168.10.1` | Succeeds (own gateway) | Pass |
| Manager PC | `ping 192.168.20.1` | Blocked | Pass |
| Manager PC | `ping 192.168.30.1` | Blocked | Pass |
| Manager PC | `ssh -l admin 192.168.99.1` | Still works (deliberate exception) | Pass |
| POS terminal | `ping 192.168.20.1` | Succeeds | Pass |
| POS terminal | `ping 192.168.10.1` | Blocked | Pass |
| POS terminal | `ping 192.168.99.1` | Blocked | Pass |
| Manager PC | `ipconfig /release`, then `/renew` | New address in 192.168.10.x | Pass |

## Inter-VLAN ACLs: management VLAN

| From | Test | Expected | Result |
|---|---|---|---|
| Manager PC | `ssh -l admin 192.168.99.2` | Still works | Pass |
| Manager PC | `ssh -l admin 192.168.99.1` | Still works | Pass |
| Manager PC | `ping 192.168.99.2` | Still works | Pass |
| Switch | `ping 192.168.99.1` | Succeeds | Pass |
| Switch | `ping 192.168.10.1` | Blocked (switch cannot initiate) | Pass |
| Switch | `ping 8.8.8.8` | Blocked | Pass |

`show access-lists` on the router shows a match counter beside each rule, which confirms which rule handled which test traffic.

## Switch hardening: unused ports, timeouts, SSH key

| Test | Expected | Result |
|---|---|---|
| `show interfaces status` | Used ports `connected`; Fa0/12-24, Gi0/2 and the reserved spares `disabled` | Pass |
| `show vlan brief` | VLAN 999 exists and holds the parked ports | Pass |
| `show running-config \| section line` | `exec-timeout 5 0` on console and VTY lines | Pass |
| Manager PC: `ssh -l admin 192.168.99.2` | Works with the regenerated 2048-bit key | Pass |
| Connectivity regression: office, POS and guest checks | Nothing broken by the hardening | Pass |

## Trunk native VLAN

| Test | Expected | Result |
|---|---|---|
| `show interfaces trunk` | Gi0/1 shows native VLAN 998; allowed list still `10,20,30,99` | To confirm |
| Manager PC: ping gateway, SSH to switch and router | Unchanged; all traffic is tagged | To confirm |
| Release and renew DHCP on a POS or guest device | Still works across the trunk | To confirm |

## Commands worth capturing as evidence

Screenshots or pasted output of these make good proof for the repo. Save them in a `docs/screenshots/` folder and link them here.

```
show ip ssh
show access-lists
show interfaces trunk
show interfaces status
show vlan brief
```
