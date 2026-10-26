# DJGC IP plan

All internal addressing is private (RFC1918). One /24 per segment; the firewall is always `.1`; DHCP clients get `.100–.200`.

## Segments

| Segment | OPNsense NIC | Subnet | Gateway (firewall) | DHCP | Purpose |
|---|---|---|---|---|---|
| WAN | hn0 | 10.0.0.0/24 (home network) | 10.0.0.1 (home router) | OPNsense WAN gets 10.0.0.41 by DHCP | Uplink, double NAT |
| SERVERS | hn1 (LAN) | 10.0.1.0/24 | 10.0.1.1 | Static only | Domain controller |
| HEADOFFICE | hn2 | 10.0.2.0/24 | 10.0.2.1 | .100–.200 via relay | Department |
| CONSTRUCTION | hn3 | 10.0.3.0/24 | 10.0.3.1 | .100–.200 via relay | Department |
| BUILDINGMAT | hn4 | 10.0.4.0/24 | 10.0.4.1 | .100–.200 via relay | Department |
| PROPERTYMGMT | hn5 | 10.0.5.0/24 | 10.0.5.1 | .100–.200 via relay | Department |
| VPN (WireGuard) | wg0 | 10.0.9.0/24 | 10.0.9.1 | Static per peer | Remote admin tunnel |

10.0.6–8 are unused and reserved for future segments. The VPN uses 10.0.9.0/24 because a subnet may exist in **one place only**. Reusing a department range or the home range would make routing ambiguous for the firewall or the laptop.

## Static addresses

| Host | IP | Notes |
|---|---|---|
| Home router | 10.0.0.1 | Upstream gateway and DHCP for WAN |
| OPNsense WAN | 10.0.0.41 | DHCP from the home router. **Should be reserved** (see baseline.md, known gaps) |
| Hyper-V host | 10.0.0.55 | Separate device from OPNsense WAN (External vSwitch, management OS not shared) |
| fw01 (OPNsense) | 10.0.1.1 … 10.0.5.1, 10.0.9.1 | `fw01.corp.djgroup.com` |
| DC01 | 10.0.1.15 | AD DS, DNS, DHCP. DNS client: itself, then 127.0.0.1 |
| VPN peer `laptop-dhruv` | 10.0.9.2/32 | One peer = one /32 |

## DHCP (on DC01)

- Four scopes, 10.0.2–5.0/24, range .100–.200.
- Options: 003 Router = segment `.1` · 006 DNS = 10.0.1.15 · 015 Domain = corp.djgroup.com.
- OPNsense **DHCP relay** on the four department interfaces → 10.0.1.15. The relay stamps the request with the interface IP (giaddr), which tells DC01 which scope to use.
- OPNsense's own DHCP (Dnsmasq) is **disabled** on SERVERS.

## DNS

- Zone `corp.djgroup.com` (AD-integrated) on DC01.
- Reverse zones for 10.0.1–5.x (secure dynamic updates).
- DC01 forwarder → 10.0.1.1 (OPNsense Unbound) → internet.
- AD Sites & Services: subnets 10.0.1–5.0/24 mapped to `Default-First-Site-Name`.

## IPv6

Disabled **at the network**: IPv6 type = None on all internal interfaces, and there are no IPv6 pass rules. Clients keep IPv6 enabled (Microsoft-supported) and only have `fe80::` / `::1`.
