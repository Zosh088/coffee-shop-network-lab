# Evidence

Command output captured from the finished lab, to back up the claims in the other docs. Router output is from `MikesCoffee-R1` and switch output is from `MikesCoffee-SW1`. The test steps behind these results are in [`verification.md`](./verification.md).

> ACL hit counters show traffic seen since the counters were last reset, so they reflect a point in time. The pass/fail results for the blocked-traffic tests are recorded in `verification.md`.

## Router

### SSH is enabled, version 2 only

```
MikesCoffee-R1#sh ip ssh
SSH Enabled - version 2.0
Authentication timeout: 120 secs; Authentication retries: 3
```

### Access lists, with match counters

What this shows: the DHCP permits are being hit (DHCP works on all three user VLANs), the office VLAN's general permit has carried traffic, and `MGMT_RESTRICTIONS` rule 10 (`established`) matched 30 packets, which is the switch's SSH replies getting back to the office. `VTY_ADMIN` has permitted office sessions and denied others.

```
MikesCoffee-R1#sh access-lists
Extended IP access list GUEST_RESTRICTIONS
    10 permit udp any eq bootpc any eq bootps (4 match(es))
    20 deny ip 192.168.30.0 0.0.0.255 192.168.10.0 0.0.0.255
    30 deny ip 192.168.30.0 0.0.0.255 192.168.20.0 0.0.0.255
    40 deny ip 192.168.30.0 0.0.0.255 192.168.99.0 0.0.0.255
    50 permit ip 192.168.30.0 0.0.0.255 any (1 match(es))
Extended IP access list OFFICE_RESTRICTIONS
    10 permit udp any eq bootpc any eq bootps (2 match(es))
    20 deny ip 192.168.10.0 0.0.0.255 192.168.20.0 0.0.0.255
    30 deny ip 192.168.10.0 0.0.0.255 192.168.30.0 0.0.0.255
    40 permit ip 192.168.10.0 0.0.0.255 any (76 match(es))
Extended IP access list POS_RESTRICTIONS
    10 permit udp any eq bootpc any eq bootps (2 match(es))
    20 deny ip 192.168.20.0 0.0.0.255 192.168.10.0 0.0.0.255
    30 deny ip 192.168.20.0 0.0.0.255 192.168.30.0 0.0.0.255
    40 deny ip 192.168.20.0 0.0.0.255 192.168.99.0 0.0.0.255
    50 permit ip 192.168.20.0 0.0.0.255 any (1 match(es))
Extended IP access list MGMT_RESTRICTIONS
    10 permit tcp 192.168.99.0 0.0.0.255 192.168.10.0 0.0.0.255 established (30 match(es))
    20 permit icmp 192.168.99.0 0.0.0.255 192.168.10.0 0.0.0.255 echo-reply
    30 permit ip 192.168.99.0 0.0.0.255 host 192.168.99.1
    40 deny ip 192.168.99.0 0.0.0.255 any
Standard IP access list VTY_ADMIN
    10 permit 192.168.10.0 0.0.0.255 (2 match(es))
    20 deny any (32 match(es))
```

## Switch

### Trunk: native VLAN 998, only four VLANs allowed

```
MikesCoffee-SW1#show interfaces trunk
Port        Mode         Encapsulation  Status        Native vlan
Gig0/1      on           802.1q         trunking      998

Port        Vlans allowed on trunk
Gig0/1      10,20,30,99

Port        Vlans allowed and active in management domain
Gig0/1      10,20,30,99

Port        Vlans in spanning tree forwarding state and not pruned
Gig0/1      10,20,30,99
```

### Port status: used ports up, everything else disabled

```
MikesCoffee-SW1#show interfaces status
Port      Name               Status       Vlan       Duplex  Speed Type
Fa0/1     ## MANAGEMENT_OFFI connected    10         a-full  a-100 10/100BaseTX
Fa0/2     ## MANAGEMENT_OFFI connected    10         a-full  a-100 10/100BaseTX
Fa0/3     ## MANAGEMENT_OFFI disabled     10         auto    auto  10/100BaseTX
Fa0/4     ## MANAGEMENT_OFFI disabled     10         auto    auto  10/100BaseTX
Fa0/5     ## MANAGEMENT_OFFI disabled     10         auto    auto  10/100BaseTX
Fa0/6     ## POS_DEVICES ##  connected    20         a-full  a-100 10/100BaseTX
Fa0/7     ## POS_DEVICES ##  connected    20         a-full  a-100 10/100BaseTX
Fa0/8     ## POS_DEVICES ##  disabled     20         auto    auto  10/100BaseTX
Fa0/9     ## POS_DEVICES ##  disabled     20         auto    auto  10/100BaseTX
Fa0/10    ## POS_DEVICES ##  disabled     20         auto    auto  10/100BaseTX
Fa0/11    ## FREEWIFI_ACCESS connected    30         a-full  a-100 10/100BaseTX
Fa0/12                       disabled     999        auto    auto  10/100BaseTX
Fa0/13                       disabled     999        auto    auto  10/100BaseTX
Fa0/14                       disabled     999        auto    auto  10/100BaseTX
Fa0/15                       disabled     999        auto    auto  10/100BaseTX
Fa0/16                       disabled     999        auto    auto  10/100BaseTX
Fa0/17                       disabled     999        auto    auto  10/100BaseTX
Fa0/18                       disabled     999        auto    auto  10/100BaseTX
Fa0/19                       disabled     999        auto    auto  10/100BaseTX
Fa0/20                       disabled     999        auto    auto  10/100BaseTX
Fa0/21                       disabled     999        auto    auto  10/100BaseTX
Fa0/22                       disabled     999        auto    auto  10/100BaseTX
Fa0/23                       disabled     999        auto    auto  10/100BaseTX
Fa0/24                       disabled     999        auto    auto  10/100BaseTX
Gig0/1    ## TO_MIKESCOFFEE_ connected    trunk      a-full  a-100 10/100/1000BaseTX
Gig0/2                       disabled     999        auto    auto  10/100/1000BaseTX
```

### VLAN database: names and port membership

VLANs 99 and 998 list no ports because they have no access ports. VLAN 99 is reached through the trunk and the SVI.

```
MikesCoffee-SW1#show vlan brief

VLAN Name                             Status    Ports
---- -------------------------------- --------- -------------------------------
1    default                          active
10   MANAGEMENT                       active    Fa0/1, Fa0/2, Fa0/3, Fa0/4
                                                Fa0/5
20   POS                              active    Fa0/6, Fa0/7, Fa0/8, Fa0/9
                                                Fa0/10
30   FREE_WIFI                        active    Fa0/11
99   NETWORK_MANAGEMENT               active
998  NATIVE_UNUSED                    active
999  PARKING_LOT                      active    Fa0/12, Fa0/13, Fa0/14, Fa0/15
                                                Fa0/16, Fa0/17, Fa0/18, Fa0/19
                                                Fa0/20, Fa0/21, Fa0/22, Fa0/23
                                                Fa0/24, Gig0/2
1002 fddi-default                     active
1003 token-ring-default               active
1004 fddinet-default                  active
1005 trnet-default                    active
```
