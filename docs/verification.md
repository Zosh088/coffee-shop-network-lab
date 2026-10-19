# Verification

How I tested each change. Every hardening step was tested before the configuration was saved and committed, so a failure could be traced to the change that caused it. Packet Tracer often times out on the first ping while ARP resolves, so each ping was repeated.

Tests written against the old shared `admin` account or the old office-to-management exception were superseded by the tests below, and have been replaced.

## Management access

| From | Test | Expected | Result |
|---|---|---|---|
| Technician laptop | `ssh -l tech1` and `ssh -l tech2` to 192.168.99.1 | Works | Pass |
| Technician laptop | `ssh -l tech1` and `ssh -l tech2` to 192.168.99.2 | Works | Pass |
| Technician laptop | `ssh -l admin` to 192.168.99.1 and 192.168.99.2 | Refused (account removed) | Pass |
| Manager PC | SSH to 192.168.99.1 and 192.168.99.2 | Refused by `VTY_ADMIN` | Pass |
| Manager PC | `ping 192.168.99.2` | Blocked | Pass |
| Guest laptop | `ssh -l tech1 192.168.30.1` | Refused or times out | Pass |
| POS terminal | `ssh -l tech1 192.168.20.1` | Refused or times out | Pass |
| Both devices | `show running-config \| section line` | `access-class VTY_ADMIN in` and `exec-timeout 5 0` on the VTY lines | Pass |

## Credentials

| Test | Expected | Result |
|---|---|---|
| `show running-config \| include username` on each device | `tech1` and `tech2` only, with no `admin` | Pass |
| `show running-config \| include enable secret` on each device | The two devices show different hashes | Pass |
| `show running-config \| section line con` on each device | `login local`, with no `password` line | Pass |
| Log out of each console and back in | Asks for a username and password; the old shared password no longer works | Pass |
| Log in as `tech1`, run `disable`, then `enable` | Asks for the new, device-specific enable secret | Pass |

## Maintenance port

| Test | Expected | Result |
|---|---|---|
| `show interfaces status` | Fa0/15 `disabled`, in VLAN 99 | Pass |
| Technician laptop while Fa0/15 is shut | No connectivity | Pass |
| `no shutdown` on Fa0/15 | Connectivity restored | Pass |
| `shutdown` on Fa0/15 when finished | Port disabled again | Pass |

## Inter-VLAN ACLs

| From | Test | Expected | Result |
|---|---|---|---|
| Manager PC | `ping 192.168.20.1` and `ping 192.168.30.1` | Blocked | Pass |
| Manager PC | `ping 192.168.99.2` | Blocked | Pass |
| POS terminal | `ping 192.168.10.1`, `192.168.30.1` and `192.168.99.1` | Blocked | Pass |
| Guest laptop | `ping 192.168.10.1`, `192.168.20.1` and `192.168.99.1` | Blocked | Pass |
| Technician laptop | `ping 192.168.99.1` | Succeeds | Pass |
| Technician laptop | `ping 192.168.10.1` | Blocked | Pass |
| Each VLAN's own gateway | Ping from a device in that VLAN | Succeeds | Pass |

## DHCP

| From | Test | Expected | Result |
|---|---|---|---|
| Manager PC | `ipconfig /release`, then `/renew` | New address in 192.168.10.x | Pass |
| POS terminal | same | New address in 192.168.20.x | Pass |
| Guest laptop | same | New address in 192.168.30.x | Pass |
| Router | `show access-lists` | The DHCP rule in each user-VLAN ACL shows matches | Pass |

## Switch hardening: unused ports, timeouts, SSH key, native VLAN

| Test | Expected | Result |
|---|---|---|
| `show interfaces status` | Used ports `connected`; unassigned ports, reserved spares and the maintenance port `disabled` | Pass |
| `show vlan brief` | VLANs 998 and 999 exist; parked ports are in VLAN 999 | Pass |
| `show interfaces trunk` | Gi0/1 shows native VLAN 998; allowed list `10,20,30,99` | Pass |
| `show running-config \| section line` | `exec-timeout 5 0` on console and VTY lines of both devices | Pass |
| SSH to both devices with the regenerated 2048-bit keys | Works | Pass |
| Connectivity regression: office, POS and guest checks | Nothing broken by the hardening | Pass |

## Evidence

Output from the key `show` commands is saved in [`evidence.md`](./evidence.md), and the full final running-configs are in [`/configs`](../configs).
