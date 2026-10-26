# DJGC lab baseline

What the lab is built from, and what state it's in when it's known-good.

## Platform

| Component | Detail |
|---|---|
| Hypervisor | Hyper-V on a Windows host (home network, 10.0.0.55) |
| External vSwitch | Bound to a physical NIC, `-AllowManagementOS $false`. Only the OPNsense WAN uses it |
| Private vSwitches | SERVERS, HEADOFFICE, CONSTRUCTION, BUILDINGMAT, PROPERTYMGMT (no uplink: the firewall is the only way out) |
| Firewall | OPNsense 26.7, VM `OPNsense01`, Gen 2, Secure Boot off, 3 GB RAM, 6 NICs (hn0–hn5) with static MACs `00:15:5D:0A:00:01–06` so interface assignment never shifts |
| Domain controller | `DC01`, Windows Server: AD DS, DNS, DHCP. Domain `corp.djgroup.com` |
| Test client | `Client01` (Windows, domain-joined), moved between department switches for testing |
| Remote client | Laptop with the WireGuard for Windows client, on the home network |

## Firewall baseline

- Default deny on every interface; least-privilege pass rules only (see `firewall-rules.md`).
- Anti-lockout **disabled**. GUI reachable only from DC01 on `SERVERS address:443`, and via the console.
- WAN: "Block private networks" **unticked**, because the WAN itself sits on a private home network (10.0.0.0/24).
- IPv6: None on all internal interfaces.
- Automatic block rules (`sshlockout`, `virusprot`) kept.

## DC01 host firewall

- Remote Desktop enabled (`fDenyTSConnections = 0`), with the Windows Firewall "Remote Desktop" rules enabled and scoped to `RemoteAddress 10.0.9.2`. This is defense in depth: even a wide OPNsense rule can't expose RDP to anyone else.

## Recovery

- **Config backup:** System → Configuration → Backups → Download, after every milestone.
- **Snapshots:** `07-wk3-dept-rules-done`; take `08-wk4-vpn-done` after this week.
- **Break-glass:** the OPNsense console (Hyper-V) with root login works even if every GUI rule is broken.

## Known gaps (accepted for a lab, would be fixed in production)

| Gap | Risk | Production fix |
|---|---|---|
| OPNsense WAN IP is DHCP (10.0.0.41) | A new lease breaks the VPN endpoint | DHCP reservation on the home router, or a static WAN IP |
| DC01 used as the admin workstation | A compromised DC also gets the firewall GUI | A dedicated admin workstation / jump box |
| Single DC | No redundancy for login, DNS, DHCP | A second DC |
| VPN reachable from the home network only | Not usable from outside yet | Port-forward UDP 51820 on the edge router, after review |
| Lab built around a single test client | Segments were tested one at a time | A client per segment, or automated tests |
