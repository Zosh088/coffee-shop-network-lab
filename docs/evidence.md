# Evidence

Command output captured from the finished lab, to back up the claims in the other docs. Router output is from `MikesCoffee-R1` and switch output is from `MikesCoffee-SW1`. The test steps behind these results are in [`verification.md`](./verification.md).

> ACL hit counters show traffic seen since the counters were last reset, so they are a snapshot in time. The pass/fail results for each test are recorded in `verification.md`.

## Router

### SSH is enabled, version 2 only

```
MikesCoffee-R1#sh ip ssh
SSH Enabled - version 2.0
Authentication timeout: 120 secs; Authentication retries: 3
```

### Access lists, with match counters (earlier capture)

Captured before administration moved to the technician laptop, so the rule numbers and the office-to-management rules have since changed. The current policy is in [`acl-notes.md`](./acl-notes.md) and the final rules are in [`configs/router1-config.txt`](../configs/router1-config.txt). This capture is kept because its hit counters show every deny rule matching blocked traffic: 8 matches on each deny line the tests were aimed at (two runs of four pings), and matches on the DHCP permits, the `established` rule and `VTY_ADMIN`.

```
MikesCoffee-R1#show access-lists
Extended IP access list GUEST_RESTRICTIONS
    10 permit udp any eq bootpc any eq bootps (4 match(es))
    20 deny ip 192.168.30.0 0.0.0.255 192.168.10.0 0.0.0.255 (8 match(es))
    30 deny ip 192.168.30.0 0.0.0.255 192.168.20.0 0.0.0.255 (8 match(es))
    40 deny ip 192.168.30.0 0.0.0.255 192.168.99.0 0.0.0.255 (8 match(es))
    50 permit ip 192.168.30.0 0.0.0.255 any (1 match(es))
Extended IP access list OFFICE_RESTRICTIONS
    10 permit udp any eq bootpc any eq bootps (2 match(es))
    20 deny ip 192.168.10.0 0.0.0.255 192.168.20.0 0.0.0.255 (8 match(es))
    30 deny ip 192.168.10.0 0.0.0.255 192.168.30.0 0.0.0.255 (8 match(es))
    40 permit ip 192.168.10.0 0.0.0.255 any (76 match(es))
Extended IP access list POS_RESTRICTIONS
    10 permit udp any eq bootpc any eq bootps (2 match(es))
    20 deny ip 192.168.20.0 0.0.0.255 192.168.10.0 0.0.0.255 (8 match(es))
    30 deny ip 192.168.20.0 0.0.0.255 192.168.30.0 0.0.0.255 (8 match(es))
    40 deny ip 192.168.20.0 0.0.0.255 192.168.99.0 0.0.0.255 (8 match(es))
    50 permit ip 192.168.20.0 0.0.0.255 any (1 match(es))
Extended IP access list MGMT_RESTRICTIONS
    10 permit tcp 192.168.99.0 0.0.0.255 192.168.10.0 0.0.0.255 established (30 match(es))
    20 permit icmp 192.168.99.0 0.0.0.255 192.168.10.0 0.0.0.255 echo-reply
    30 permit ip 192.168.99.0 0.0.0.255 host 192.168.99.1
    40 deny ip 192.168.99.0 0.0.0.255 any (9 match(es))
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

The maintenance port (Fa0/15, VLAN 99) is disabled; it is enabled only during maintenance.

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
Fa0/15    ## TECH_MAINTENANC disabled     99         auto    auto  10/100BaseTX
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

VLAN 99 now lists Fa0/15, the maintenance port, and the parked ports are in VLAN 999. VLAN 998 lists no ports because nothing is assigned to it.

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
99   NETWORK_MANAGEMENT               active    Fa0/15
998  NATIVE_UNUSED                    active
999  PARKING_LOT                      active    Fa0/12, Fa0/13, Fa0/14, Fa0/16
                                                Fa0/17, Fa0/18, Fa0/19, Fa0/20
                                                Fa0/21, Fa0/22, Fa0/23, Fa0/24
                                                Gig0/2
1002 fddi-default                     active
1003 token-ring-default               active
1004 fddinet-default                  active
1005 trnet-default                    active
```

### Port security: summary

Captured after the tests had finished and the rogue device was removed, so the violation counts have reset to 0. The two per-interface captures further down were taken during the tests.

```
MikesCoffee-SW1#show port-security
Secure Port  MaxSecureAddr  CurrentAddr  SecurityViolation  Security Action
                (Count)       (Count)          (Count)
--------------------------------------------------------------------
      Fa0/1        1             1                0            Restrict
      Fa0/2        1             1                0            Restrict
      Fa0/6        1             1                0            Restrict
      Fa0/7        1             1                0            Restrict
     Fa0/15        1             1                0            Shutdown
----------------------------------------------------------------------
```

### Port security: learned and configured addresses

The four office and POS ports each learned one sticky address. The maintenance port holds the technician laptop's address, which was configured by hand.

```
MikesCoffee-SW1#show port-security address
          Secure Mac Address Table
-----------------------------------------------------------------------------
Vlan    Mac Address       Type                Ports    Remaining Age
                                                           (mins)
----    -----------       ----                -----    -------------
  10    0003.E4C2.EB45    SecureSticky        Fa0/1        -
  10    0007.EC13.76A7    SecureSticky        Fa0/2        -
  20    000C.CF4D.CAE3    SecureSticky        Fa0/6        -
  20    0090.2B78.4BBE    SecureSticky        Fa0/7        -
  99    0001.96B4.C891    SecureConfigured    Fa0/15       -
-----------------------------------------------------------------------------
Total Addresses in System (excluding one mac per port)     : 0
Max Addresses limit in System (excluding one mac per port) : 1024
```

### Port security: restrict mode with a rogue device on a POS port

The port stays `Secure-up` while the stranger's frames are dropped and counted. The last source address is the rogue PC, the same address that appears in the maintenance-port test below.

```
MikesCoffee-SW1#show port-security interface f0/6
Port Security              : Enabled
Port Status                : Secure-up
Violation Mode             : Restrict
Aging Time                 : 0 mins
Aging Type                 : Absolute
SecureStatic Address Aging : Disabled
Maximum MAC Addresses      : 1
Total MAC Addresses        : 1
Configured MAC Addresses   : 0
Sticky MAC Addresses       : 1
Last Source Address:Vlan   : 00E0.F987.31B5:20
Security Violation Count   : 6
```

### Port security: shutdown mode with a rogue device on the maintenance port

The first violation shut the port down. It stays down until someone runs `shutdown` and `no shutdown`.

```
MikesCoffee-SW1#show port-security interface f0/15
Port Security              : Enabled
Port Status                : Secure-shutdown
Violation Mode             : Shutdown
Aging Time                 : 0 mins
Aging Type                 : Absolute
SecureStatic Address Aging : Disabled
Maximum MAC Addresses      : 1
Total MAC Addresses        : 1
Configured MAC Addresses   : 1
Sticky MAC Addresses       : 0
Last Source Address:Vlan   : 00E0.F987.31B5:99
Security Violation Count   : 1
```

### Management ACL on the switch

```
MikesCoffee-SW1#show access-lists
Standard IP access list VTY_ADMIN
    10 permit host 192.168.99.10
    20 deny any
```
