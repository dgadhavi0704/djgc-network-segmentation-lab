# DJGC firewall rules (OPNsense)

Policy: **least privilege + default deny.** Each segment gets only what its job needs. Pass rules only pass what matches; everything else falls to the implicit default deny at the bottom of each interface. There are no "block department X" rules because none are needed.

Rules are evaluated **inbound on the interface closest to the sender**, top to bottom, **first match wins**. Replies return via the **state table** (no rule needed).

## Aliases

| Alias | Type | Content | Purpose |
|---|---|---|---|
| `AD_DC_Ports` | Port(s) | 53, 88, 123, 135, 389, 445, 464, 636, 3268-3269, 49152-65535 | What a domain client needs from DC01 |
| `Web_Ports` | Port(s) | 80, 443 | Web browsing: HTTP (plain web), HTTPS (encrypted web) |
| `RFC1918_Private` | Network(s) | 10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16 | All private ranges; inverted to mean "internet only" |

Port reference: 53 DNS (name lookup) · 88 Kerberos (login tickets) · 123 NTP (time sync) · 135 RPC (endpoint mapper) · 389 LDAP (directory lookups) · 445 SMB (file shares, GPO) · 464 Kerberos password change · 636 LDAPS (secure LDAP) · 3268-3269 Global Catalog (forest search) · 49152-65535 RPC dynamic (AD replies).

> Always verify an alias under **Firewall → Diagnostics → Aliases**. What the firewall loaded is what counts, not the edit screen. (See break/fix #4.)

## SERVERS (hn1, 10.0.1.0/24, gateway 10.0.1.1)

### DC01 -> firewall DNS forwarder
- **Rule:** Pass · in · SERVERS · TCP/UDP · src 10.0.1.15 · dst 10.0.1.1 · port 53 (DNS)
- **Why:** DC01 forwards internet name lookups to the firewall. Nothing else on SERVERS needs it.
- **Tested:** `Resolve-DnsName google.com` on DC01 worked; Live View showed this rule passing.

### DC01 -> internet HTTPS
- **Rule:** Pass · in · SERVERS · TCP · src 10.0.1.15 · dst !RFC1918_Private · port 443 (HTTPS)
- **Why:** DC01's own updates and Microsoft services. It does not carry client browsing.
- **Tested:** HTTPS to the internet worked. This rule had the empty-alias problem (break/fix #4) and was re-checked after the fix.

### DC01 -> firewall admin GUI
- **Rule:** Pass · in · SERVERS · TCP · src 10.0.1.15 · dst **SERVERS address** · port 443 (HTTPS)
- **Why:** Only one admin machine may reach the firewall's GUI. Anti-lockout (which let all of SERVERS in) is disabled. `SERVERS address` = only the firewall's leg on this segment, and it follows the IP if it changes.
- **Note:** A DC is not a good admin workstation. This is a lab compromise until a dedicated admin box exists.
- **Tested:** ALLOW `Test-NetConnection 10.0.1.1 -Port 443` = True. BLOCK `Test-NetConnection 10.0.3.1 -Port 443` = False.
- **Break-glass:** OPNsense console (Hyper-V) root login works even if all GUI rules break.

### Disabled
- Default "allow LAN to any" (IPv4 + IPv6): disabled. Replaced by the rules above.
- Anti-lockout: disabled (System → Settings → Administration), only after the GUI rule was proven.

## Departments: HEADOFFICE, CONSTRUCTION, BUILDINGMAT, PROPERTYMGMT

Each department interface has the same two rules; only the interface and source change.

| Interface | Subnet | Gateway |
|---|---|---|
| HEADOFFICE (hn2) | 10.0.2.0/24 | 10.0.2.1 |
| CONSTRUCTION (hn3) | 10.0.3.0/24 | 10.0.3.1 |
| BUILDINGMAT (hn4) | 10.0.4.0/24 | 10.0.4.1 |
| PROPERTYMGMT (hn5) | 10.0.5.0/24 | 10.0.5.1 |

### <DEPT> clients -> DC01 (domain services)
- **Rule:** Pass · in · <DEPT> · TCP/UDP · src <DEPT> net · dst 10.0.1.15 · port `AD_DC_Ports`
- **Why:** Login, Group Policy and DNS all come from DC01. Without it there's no domain login and no name resolution, so no browsing either.
- **Note:** 443 to DC01 is intentionally not allowed.

### <DEPT> clients -> internet web only
- **Rule:** Pass · in · <DEPT> · TCP · src <DEPT> net · dst !RFC1918_Private (inverted) · port `Web_Ports`
- **Why:** Users need websites, not other departments. The rule excludes private ranges, so internal traffic never matches it and is dropped by default deny.
- **Path:** client -> firewall -> WAN -> internet; replies return via the state table. Only the DNS lookup goes to DC01. DC01 is not a proxy.
- **Tested:** see the test matrix.

### DHCP
- DHCP relay on all four department interfaces → DC01 (10.0.1.15). Scopes .100-.200 per subnet.
- Observed: relayed DHCP worked without a pass rule for 67 (DHCP server). The relay runs on the firewall itself.

## WAN

### Home LAN -> WireGuard
- **Rule:** Pass · in · WAN · UDP · src 10.0.0.0/24 · dst WAN address · port 51820 (WireGuard, VPN tunnel)
- **Why:** The only door from outside. Lets in the VPN's outer envelope, from the home network only for now.
- **Tested:** Live View showed 10.0.0.8 -> 10.0.0.41:51820 passing; the peer completed its handshake.

## WireGuard (Group): remote admin VPN (10.0.9.0/24)

Tunnel traffic is unwrapped and then checked here, like any other segment. Source is the **single peer IP**, so a future second peer gets nothing by default. Full build in `vpn-runbook.md`.

### VPN laptop -> DC01 RDP
- **Rule:** Pass · in · WireGuard · TCP · src 10.0.9.2 · dst 10.0.1.15 · port 3389 (RDP, remote desktop) · Log
- **Why:** Daily remote administration of DC01.
- **Tested:** `Test-NetConnection 10.0.1.15 -Port 3389` = True; `mstsc` session opened; Live View showed this label on wg0 in.

### VPN laptop -> DC01 DNS
- **Rule:** Pass · in · WireGuard · TCP/UDP · src 10.0.9.2 · dst 10.0.1.15 · port 53 (DNS, name lookup) · Log
- **Why:** Resolve lab names such as `dc01.corp.djgroup.com`.
- **Tested:** Live View showed UDP 53 passing with this label after the port fix (break/fix #13).

### Not allowed (default deny)
SMB 445, NetBIOS 137, ICMP, and the firewall GUI (10.0.9.1:443). The temporary ICMP test rule was deleted after the first test.

## IPv6

- IPv6 Configuration Type = **None** on all five internal interfaces (SERVERS was "Identity Association" from the wizard; changed).
- No IPv6 pass rules. Clients keep IPv6 enabled (Microsoft-supported) but only have `fe80::` (link-local) and `::1` (loopback).
- **Why:** every rule here is IPv4. An IPv6 path would bypass all of them.

## Automatic rules kept
- `sshlockout` / `virusprot` (block): brute-force protection. Keep.
- "Let out anything from firewall": the firewall's own outbound (e.g., DNS forwarding).

## Test matrix

Method: test with the **same protocol and port** the rule allows, and always include a test that **should fail**. Ping (ICMP) does not test a TCP rule.

| Segment | Domain login (`nltest /sc_verify`) | Internet (`Test-NetConnection google.com -Port 443`) | Blocked: other dept on 443 |
|---|---|---|---|
| HEADOFFICE | ✓ | ✓ | ✓ blocked |
| CONSTRUCTION | ✓ | ✓ | ✓ blocked (10.0.4.1 after alias fix) |
| BUILDINGMAT | ✓ | ✓ | ✓ blocked (10.0.2.1) |
| PROPERTYMGMT | ✓ | ✓ | ✓ blocked |
| SERVERS (DC01) | DNS, gpupdate, DHCP renew ✓ | ✓ | GUI via 10.0.3.1 ✓ blocked |
| IPv6 (client) | only fe80 / ::1 | no IPv6 route | n/a |

### VPN (laptop 10.0.0.8 via tunnel, as 10.0.9.2)

| Test | Expected | Result |
|---|---|---|
| RDP 10.0.1.15:3389 | allow | ✓ passed (`VPN- Laptop -> DC01 RDP`) |
| DNS 10.0.1.15:53 | allow | ✓ passed (`VPN- Laptop -> DC01 DNS`) |
| SMB 10.0.1.15:445 | block | ✓ blocked, default deny on wg0 |
| NetBIOS 10.0.1.15:137 | block | ✓ blocked, default deny on wg0 |
| ICMP to DC01 | block | ✓ blocked, default deny on wg0 |
| Firewall GUI 10.0.9.1:443 | block | ✓ blocked |

Config backup downloaded and snapshot `07-wk3-dept-rules-done` taken after the Week 3 tests; take `08-wk4-vpn-done` after the VPN tests.
