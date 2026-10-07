# Security Hardening

## Device Access Control

| Device | Console Password | Privileged EXEC | Remote Access |
|---|---|---|---|
| MikesCoffee-SW1 | Set, `logging synchronous` enabled | `enable secret` (MD5-hashed) | SSH only (`transport input ssh`), local user DB |
| MikesCoffee-R1 | Set, `logging synchronous` enabled | `enable secret` (MD5-hashed) | See note below |

- `service password-encryption` is enabled on both devices, so any plaintext passwords in the config (e.g. line passwords) are stored as reversible Type 7 encryption rather than cleartext — better than nothing, but not cryptographically secure. `enable secret` uses a proper one-way hash (MD5, shown as `$1$...` in the running-config) instead.
- `no ip domain-lookup` is set on both devices — this stops the CLI from trying to resolve mistyped commands as DNS hostnames, which otherwise causes an annoying delay on every typo.
- A `banner motd` warning ("Unauthorized Access is Strictly Prohibited.") is configured on both devices, displayed before login.

### SSH (Switch)

- `username admin privilege 15 secret <hash>` — local authentication, full privilege
- `ip domain-name zennstudies.local` set (required before RSA key generation)
- `crypto key generate rsa` run to enable SSH
- `ip ssh version 2` — forces SSHv2 only
- VTY lines 0–15: `login local` + `transport input ssh` — Telnet is disabled entirely, only SSH with local credentials is accepted
- `exec-timeout 10` on the management SVI's line config — idle sessions are dropped after 10 minutes

### Router remote access — open item

The router's VTY lines (`line vty 0 4`) currently only have `login` configured, with **no password and no `transport input ssh`** set. As-is, this leaves remote line access to the router incomplete — with `login` set but no password, IOS will actually refuse remote logins ("password required but none set") rather than allow unauthenticated access, but SSH was never explicitly enabled on the router the way it was on the switch. This is a good candidate for the "Next Steps" list: mirror the switch's SSH configuration (local user, `ip ssh version 2`, `transport input ssh`) onto the router so it can be managed the same way.

## Layer 2 Protections

- **PortFast** is enabled on every access port (Fa0/1–Fa0/11). Since these ports connect only to end devices (PCs, printers, an access point) and never to another switch, PortFast skips the normal STP listening/learning delay so the port comes up immediately instead of taking ~30 seconds.
- **BPDU Guard** is enabled alongside PortFast on the same ports. If a port configured for PortFast ever receives a BPDU (which would only happen if someone plugged in an unauthorized switch or hub), BPDU Guard immediately puts the port into `err-disabled` state — protecting the network from accidental or malicious topology changes on what should be an end-device-only port.
- The trunk to the router (`Gi0/1`) is explicitly set to `switchport mode trunk` with `switchport trunk allowed vlan 10,20,30,99` — only the VLANs actually in use are permitted across the trunk, rather than allowing all VLANs by default.

## Layer 3 Protections

See [`acl-notes.md`](./acl-notes.md) for the guest network isolation ACL.

## What I'd Harden Further

- Finish SSH configuration on the router (see open item above)
- Move from a shared local `admin` account to per-technician accounts for accountability
- Consider AAA (e.g. local RADIUS/TACACS+ simulation) if extending this lab further
- Add `service password-encryption` verification and periodically rotate the `enable secret`
