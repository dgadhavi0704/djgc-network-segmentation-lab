# DJGC remote access VPN runbook (WireGuard)

Remote admin access from a laptop into the lab, with least privilege: the VPN is just one more segment behind default deny.

## How it works

The laptop wraps an encrypted **inner packet** (10.0.9.2 → DC01) inside an **outer packet** (laptop → OPNsense WAN 10.0.0.41 : UDP 51820). OPNsense unwraps it, then checks the inner packet against the **WireGuard** rules like any other interface. DC01 only ever sees source **10.0.9.2**. There's no NAT inside the tunnel.

| Kind of IP | Value | Config field |
|---|---|---|
| Real (outer envelope) | Laptop 10.0.0.x → WAN 10.0.0.41 | Laptop `Endpoint` |
| Tunnel (inner identity) | Firewall 10.0.9.1, laptop 10.0.9.2 | `Tunnel address` / `Address` |
| Behind the tunnel | SERVERS 10.0.1.0/24 | Laptop `AllowedIPs` (split tunnel) |

**Keys:** each side keeps its **private** key and gives the other its **public** key. WireGuard never answers a packet whose key it doesn't know, so to anyone else the port looks closed.

## Build

### 1. OPNsense instance
VPN → WireGuard → Instances → +
- Name `wg-remote` · generate key pair (gear icon) · Listen port `51820` · Tunnel address `10.0.9.1/24`

### 2. Laptop client
```powershell
# Install the WireGuard client
winget install WireGuard.WireGuard
```
Add Tunnel → Add empty tunnel → name `djgc` (this generates the laptop's keys). Copy its **public** key.
```ini
[Interface]
PrivateKey = <auto, never leaves the laptop>
Address = 10.0.9.2/32

[Peer]
PublicKey = <OPNsense instance public key>
Endpoint = 10.0.0.41:51820      # OPNsense WAN, NOT the Hyper-V host
AllowedIPs = 10.0.1.0/24, 10.0.9.0/24   # only lab traffic goes in the tunnel
```

### 3. OPNsense peer
VPN → WireGuard → Peers → +
- Name `laptop-dhruv` · Public key = **laptop's** public key · Allowed IPs `10.0.9.2/32` · Instance `wg-remote` · Endpoint blank (learned from the handshake)
- General → **Enable WireGuard** → Apply

One peer = one /32. On the firewall, a peer's Allowed IPs decides both **which source IPs it may use** and **where replies go**. A /24 on one peer would let it impersonate others and steal their replies.

### 4. Firewall rules

**WAN** (the door for the outer envelope):

| Proto | Source | Destination | Port | Description |
|---|---|---|---|---|
| UDP | 10.0.0.0/24 | WAN address | 51820 (WireGuard) | `Home LAN - Wireguard (Labvpn)` |

**WireGuard (Group)** (what the tunnel may do), Log ticked on each:

| Proto | Source | Destination | Port | Description |
|---|---|---|---|---|
| TCP | 10.0.9.2 | 10.0.1.15 | 3389 (RDP, remote desktop) | `VPN- Laptop -> DC01 RDP` |
| TCP/UDP | 10.0.9.2 | 10.0.1.15 | 53 (DNS, name lookup) | `VPN- Laptop -> DC01 DNS` |

Everything else (SMB 445, NetBIOS 137, ICMP, the firewall GUI) hits default deny.

### 5. DC01 (second firewall in the path)
```powershell
# Enable Remote Desktop (0 = allow)
Set-ItemProperty 'HKLM:\System\CurrentControlSet\Control\Terminal Server' -Name fDenyTSConnections -Value 0
# Enable the built-in RDP firewall rules
Enable-NetFirewallRule -DisplayGroup "Remote Desktop"
# Only the VPN laptop may connect (defense in depth)
Set-NetFirewallRule -DisplayGroup "Remote Desktop" -RemoteAddress 10.0.9.2
```

## Daily use
1. Open WireGuard on the laptop and **Activate** `djgc`.
2. `mstsc /v:10.0.1.15`, then log in as `CORP\<admin>` (DOMAIN\user, because the laptop isn't domain-joined).
3. **Deactivate** when done.

## Verify
```powershell
Test-NetConnection 10.0.1.15 -Port 3389                     # ALLOW: True
Resolve-DnsName dc01.corp.djgroup.com -Server 10.0.1.15     # ALLOW: returns 10.0.1.15
Test-NetConnection 10.0.1.15 -Port 445                      # BLOCK: False
Test-NetConnection 10.0.9.1 -Port 443                       # BLOCK: False
```
OPNsense: VPN → WireGuard → Status. The peer shows a recent **handshake** and an endpoint of `10.0.0.x:<port>`.

## Troubleshooting: find where it dies

| Symptom | Where it dies | Check |
|---|---|---|
| Peer has no handshake, and Live View shows nothing on 51820 | Before the firewall | Is there traffic? WireGuard only handshakes when something is sent (`ping 10.0.9.1`). Is the Endpoint the **OPNsense WAN**, not the host? Use Packet Capture on WAN, UDP 51820 |
| 51820 shows **blocked** on WAN | WAN rule | WAN rule source/port; "Block private networks" unticked |
| 51820 **passes** but there's no handshake | WireGuard | Key mismatch: compare the first 6 / last 4 characters on both sides; peer linked to the instance; Apply clicked |
| Handshake OK, but traffic gets no reply (Sent ≪ Received) | WireGuard rules | Live View, src 10.0.9.2: a red default deny on wg0 means a missing rule |
| wg0 in **passes** but the test still fails | Destination host | The service is running? The host firewall allows it? (DC01: RDP off and firewall rules disabled) |
| Something passes that shouldn't | A rule too wide | Check the passing rule's **label**; look for port "any" or an empty alias |

A pass in the firewall log proves the **network path**, not that the **service** answered.

## Known limitations
- WAN IP is DHCP (10.0.0.41). Reserve it on the home router for reliable daily use.
- Reachable from the home network only. Outside access needs UDP 51820 forwarded on the edge router.
- Split tunnel: only 10.0.1.0/24 and 10.0.9.0/24 use the VPN. Add department subnets to `AllowedIPs` **and** rules only if a real task needs them.
