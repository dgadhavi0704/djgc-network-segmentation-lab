# DJGC break/fix journal

Format: **Symptom → Cause → Fix → Lesson.** Real issues hit while building the lab, not staged ones.

---

### 1. Client locked out after moving networks (Week 2)
- **Symptom:** Client01 couldn't log in. It had a leftover static IP (10.0.0.65), the local admin password was unknown, and the standard user couldn't elevate.
- **Cause:** The client was built on the old flat network; the static IP no longer matched any segment, so there was no DC and no domain login.
- **Fix:** Temporary second IP on DC01 (10.0.0.15) and the client on the SERVERS switch to reach the DC → elevated, switched the NIC to DHCP, reset the local admin. Temporary IP removed afterwards.
- **Lesson:** Before moving a machine between networks, have a known local admin (LAPS / break-glass). Clean up temporary bridges.

### 2. LDAP rule never matched (Week 2)
- **Symptom:** Domain lookups failed even though an LDAP rule existed.
- **Cause:** The rule had **source port 389**. Clients connect *from* a random high (ephemeral) port *to* 389.
- **Fix:** Source port = any, destination port = 389. Later replaced by the `AD_DC_Ports` alias.
- **Lesson:** The service port is almost always the **destination** port.

### 3. Unexpected DHCP on SERVERS (Week 2)
- **Symptom:** Found the firewall's own DHCP (Dnsmasq) active on SERVERS.
- **Cause:** Wizard default.
- **Fix:** Disabled. DC01 is the only DHCP server; departments use relay.
- **Lesson:** Two DHCP servers on one segment = random leases. Check defaults after any wizard.

### 4. "Web only" rule allowed everything (Week 3)
- **Symptom:** From CONSTRUCTION, `Test-NetConnection 10.0.4.1 -Port 443` succeeded and reached the firewall GUI, passed by the web rule with Invert ticked.
- **Cause:** `RFC1918_Private` was **loaded empty** (Diagnostics → Aliases showed nothing). NOT(empty) = every address, so the rule acted as allow-any on 80/443. The same hole existed on DC01's 443 rule.
- **Fix:** Set alias type to **Network(s)**, applied, confirmed 3 entries loaded. Retested: 10.0.4.1:443 blocked, google:443 allowed.
- **Lesson:** Verify aliases in Diagnostics, not the edit screen. The earlier test used **ping**, which is ICMP and never touched the TCP 443 rule. A clue was missed too: DC01 was seen reaching 10.0.1.1:443.

### 5. Whole SERVERS segment could open the firewall GUI (Week 3)
- **Symptom:** DC01 reached the GUI with no rule of mine allowing it.
- **Cause:** The **anti-lockout** rule on the LAN interface (SERVERS) lets the whole LAN reach the GUI (443) and SSH (22).
- **Fix:** Created DC01 → SERVERS address:443, proved it in Live View, **then** disabled anti-lockout.
- **Lesson:** Build and test the replacement before removing a safety rule. Keep console access as break-glass.

### 6. IPv6 disabled on the client NIC (Week 3)
- **Symptom:** Client01 had no IPv6 address at all, not even fe80.
- **Cause:** IPv6 had been unbound on the NIC earlier.
- **Fix:** `Enable-NetAdapterBinding -ComponentID ms_tcpip6`; IPv6 blocking moved to the firewall.
- **Lesson:** Microsoft doesn't support disabling IPv6 on domain machines. Control it at the network, not the client.

### 7. Latent IPv6 path on SERVERS (Week 3)
- **Symptom:** SERVERS IPv6 type was "Identity Association" (prefix delegation from WAN).
- **Cause:** Wizard default on the LAN interface.
- **Fix:** Set to None on all five interfaces. DC01 had only fe80 / ::1 before and after: upstream never delegated a prefix.
- **Lesson:** The gap was latent. If upstream ever delegated, DC01 would get a public IPv6 address that every IPv4 rule ignores. Close it by design, not luck.

### 8. Two false results while testing (Week 3)
- **`Select-Object ipadress`** (typo) showed an empty column instead of an error, hiding the fe80 address. PowerShell doesn't complain about a property that doesn't exist.
- **`Test-NetConnection ipv6.google.com`** failed at **name resolution**, not at the firewall. Windows skips AAAA lookups when it only has link-local IPv6, so the test proved nothing about the firewall.
- **Lesson:** Read *why* a test failed before counting it as proof.

---

## Week 4: WireGuard VPN

### 9. Tunnel active, but zero packets at the firewall
- **Symptom:** The laptop's tunnel showed active; OPNsense showed no handshake, Received 0; nothing in Live View on 51820.
- **Cause:** WireGuard only handshakes when there's traffic to send. Activating the tunnel just builds the adapter.
- **Fix:** Sent traffic into the tunnel (`ping 10.0.9.1`). Packets then appeared on WAN.
- **Lesson:** "Nothing in the logs" can mean nothing was sent. Generate traffic before judging a tunnel.

### 10. Endpoint pointed at the Hyper-V host
- **Symptom:** No packets reached the OPNsense WAN, even in a packet capture.
- **Cause:** The laptop's `Endpoint` was set to the Hyper-V host's IP (10.0.0.55), not the OPNsense WAN (10.0.0.41). With `-AllowManagementOS $false` they're two separate devices; the host doesn't run WireGuard and silently dropped the packets.
- **Fix:** `Endpoint = 10.0.0.41:51820` (from Interfaces → Overview → WAN).
- **Lesson:** The host is the building, and the firewall's WAN NIC is its own device with its own IP. **Packet capture** sees packets before any rule. Use it to answer "did it even arrive?"

### 11. Packets passed WAN, but no handshake
- **Symptom:** Live View showed 10.0.0.8 -> 10.0.0.41:51820 **pass**, but the peer stayed red with no handshake.
- **Cause:** A key mismatch between the peer and the instance. WireGuard silently ignores packets it can't authenticate and never replies.
- **Fix:** Re-pasted both public keys (laptop public key → OPNsense peer; instance public key → laptop `[Peer]`). Handshake succeeded and the endpoint 10.0.0.8:<port> was learned.
- **Lesson:** "Passed the firewall, nothing happens" on a VPN = keys. Compare the first 6 / last 4 characters on both sides. Never screenshot a private key.

### 12. Green on OPNsense, but RDP still failed
- **Symptom:** Live View showed 10.0.9.2 -> 10.0.1.15:3389 **passing** (out on SERVERS), yet `Test-NetConnection -Port 3389` = False.
- **Cause:** A second firewall in the path. On DC01, Remote Desktop was off (`fDenyTSConnections = 1`, nothing listening on 3389) and the Windows Firewall RDP rules were disabled.
- **Fix:** Enabled RDP, enabled the RDP firewall rules, scoped them to `RemoteAddress 10.0.9.2`. The test returned True and the `mstsc` session opened.
- **Lesson:** A pass in the firewall log proves the **network path**, not that the **service** answered. Next stop is always the destination host: is the service running, and does the host firewall allow it? Also, the in line on wg0 was missing at first because the rule wasn't logging. **No log line doesn't mean no traffic.**

### 13. "DNS" rule allowed every port on DC01
- **Symptom:** The block test `Test-NetConnection 10.0.1.15 -Port 445` (SMB) **succeeded**. Live View: wg0 in, pass, label `VPN- Laptop -> DC01 DNS`. NetBIOS 137 was passing under the same label.
- **Cause:** The DNS rule's destination port was **any**. It was labeled DNS but acted as allow-all to DC01.
- **Fix:** Destination port set to 53. Retest: 445, 137 and ICMP blocked by default deny; DNS 53 still passing.
- **Lesson:** A label is a promise, and the port field is the truth. Only a test that **should fail** finds this. It's the same pattern as #4 (empty alias): the rule allowed more than it said.
