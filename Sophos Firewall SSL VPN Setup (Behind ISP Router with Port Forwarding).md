# Sophos Firewall SSL VPN Setup (Behind ISP Router with Port Forwarding)

Documentation for configuring and troubleshooting a remote-access SSL VPN on a Sophos Firewall (SFOS) that sits behind an ISP-provided router (port-forward scenario) with a static public IP and a DDNS hostname.

> All hostnames, IPs, and organisation names below are placeholders. Replace `xxx` values with your own.

---

## 1. Environment

| Item | Value |
|---|---|
| Firewall | Sophos Firewall (SFOS) |
| ISP | Malaysian ISP, router in router mode (NAT) |
| Public IP | Static, `xxx.xxx.xxx.xxx` |
| DDNS hostname | `vpn.xxx.example` |
| Sophos WAN IP | Private, `192.168.x.x` (behind ISP router) |
| Sophos LAN | `192.168.y.0/24` |
| VPN protocol / port | TCP / 8443 |
| VPN lease range | `10.81.234.0/24` |
| VPN tunnel mode | Default gateway (full tunnel) |

### Topology

```
Remote client
     |
  Internet
     |
[ISP router]  Public static IP, port forward TCP 8443 -> Sophos WAN IP
     |
[Sophos WAN: 192.168.x.x]
[Sophos LAN: 192.168.y.1]
     |
LAN servers / other firewalls
```

---

## 2. Prerequisites

1. **The ISP router's WAN IP must be the real public static IP.**
   - Compare the router's WAN IP with the result of whatismyip.com from a LAN PC.
   - If the router shows `100.64.x.x`-`100.127.x.x`, it is behind carrier-grade NAT and inbound VPN cannot work. Contact the ISP.
2. **The Sophos WAN IP must be stable.** Use a DHCP reservation on the ISP router, or a static private IP outside the DHCP pool.
3. **A DDNS hostname** (optional with a static IP, but convenient) that resolves to the public IP.
4. **VPN users** created under Authentication > Users, optionally in a group.

---

## 3. ISP Router Configuration

1. Create a **port forward** (virtual server):
   - Protocol: TCP
   - External port: 8443
   - Internal IP: Sophos WAN IP
   - Internal port: 8443
2. Alternative: set the Sophos WAN IP as the router's **DMZ host**. Port forwarding is preferred because it exposes less.
3. Best option if the ISP allows it: **bridge mode**, so the Sophos holds the public IP directly and no forward is needed.
4. If the router has a built-in **DDNS client** section, configure it with the DDNS provider's details (hostname, username, password/token). Setting the DDNS in the router's network/DDNS settings was required in this deployment for the hostname to work correctly.

---

## 4. Sophos Configuration

### 4.1 Allow SSL VPN on the WAN zone

**Administration > Device access**

- WAN zone: tick **SSL VPN**.
- VPN zone: tick **PING** (so the firewall's LAN IP answers pings through the tunnel).
- Only tick **User portal** on WAN if remote users must download the client from outside. Protect it with MFA if you do.

### 4.2 SSL VPN global settings

**Configure > Remote access VPN > SSL VPN > Global settings**

| Field | Value | Notes |
|---|---|---|
| Protocol | TCP | UDP gives better performance if the ISP path allows it |
| Port | 8443 | Avoid 443 if the User Portal / WebAdmin uses it |
| SSL server certificate | ApplianceCertificate | Self-signed by default; use a trusted cert to avoid client warnings |
| **Override hostname** | `vpn.xxx.example` | **Critical.** Must be the public hostname or IP |
| IPv4 lease range | `10.81.234.0/24` | Must not overlap the LAN or any client home network |
| Lease mode | IPv4 only | |
| IPv4 DNS | Internal DNS server or `1.1.1.1` / `8.8.8.8` | Blank DNS with full tunnel gives "connected but nothing loads" |
| Disconnect dead peer after | 180 s | |

> **Why Override hostname matters:** when it is blank, Sophos writes its own WAN interface IP into the client configuration. Behind NAT that is a private IP, so the VPN connects from inside the LAN but times out from the internet.

### 4.3 SSL VPN policy

**Configure > Remote access VPN > SSL VPN > Add**

- Name: `xxx-VPN`
- Policy members: the VPN users or group (an "Open Group" only works if the users are members of it)
- Tunnel access: **Use as default gateway** (full tunnel), or list the internal networks under *Permitted network resources* for split tunnel

### 4.4 Firewall rules

**Rules and policies > Firewall rules**

| Rule | Source zone | Source network | Dest zone | Dest network | Service | Action | NAT |
|---|---|---|---|---|---|---|---|
| VPN to LAN | VPN | `10.81.234.0/24` (or `##ALL_SSLVPN_RW`) | LAN (and any server zones) | Internal subnets | Any (restrict later) | Accept | Optional masquerade |
| VPN to WAN (full tunnel only) | VPN | `10.81.234.0/24` | WAN | Any | Any | Accept | Masquerade |

Place these rules **above** any broader deny rules, as the first match wins.

### 4.5 Return routing

Servers see VPN traffic from `10.81.234.x` and reply to their default gateway.

- If the Sophos LAN IP is the gateway: nothing to do.
- If another router or firewall is the gateway: add a static route `10.81.234.0/24 -> Sophos LAN IP` on it.
- Quick alternative: enable **masquerade** on the VPN-to-LAN rule so servers see the Sophos LAN IP as the source.

---

## 5. Client Setup

1. Log in to the User Portal from inside the LAN (`https://<Sophos-LAN-IP>`).
2. Download the **Sophos Connect client** and its configuration, or the OpenVPN `.ovpn` file.
3. **Always download a fresh config after changing Override hostname.** Old files keep the old address.
4. Verify the `remote` line in the config reads `vpn.xxx.example 8443`.
5. Import the config, connect, and accept the certificate warning if using the self-signed certificate.

---

## 6. Testing

Always test from a **truly external network** (mobile hotspot). Testing from the LAN gives misleading results because of NAT loopback.

```powershell
# Resolve DDNS and compare to the public IP
nslookup vpn.xxx.example

# Test TCP reachability of the VPN port
Test-NetConnection vpn.xxx.example -Port 8443
```

```bash
# Linux / macOS
nc -vz vpn.xxx.example 8443
```

After connecting:

```powershell
ping <Sophos-LAN-IP>
ping <server-IP>
Test-NetConnection <server-IP> -Port 3389
route print
```

---

## 7. Troubleshooting Guide

### 7.1 Cannot connect from the internet (works from the LAN)

| Check | What to look for | Fix |
|---|---|---|
| Client config `remote` line | Shows a private IP | Set Override hostname, re-download config |
| DDNS resolution | `nslookup` result differs from the public IP | Fix the record at the DDNS provider / router DDNS settings |
| Router WAN IP | Differs from the public IP, or is `100.64.x.x` | Contact the ISP to confirm static IP routing |
| Port forward | Wrong target IP or port | Forward TCP 8443 to the current Sophos WAN IP |
| Device access | SSL VPN not ticked for WAN | Tick it |
| Port test from outside | Timeout | Check Sophos **Log viewer > Firewall**: packets arriving and dropped means Device access; no packets means router or ISP block. Try port 4433 or 10443 |

### 7.2 Connected but cannot reach anything

1. **Firewall rule:** missing, wrong zone, or below another matching rule. Check Log viewer > Firewall while pinging.
2. **Device access:** PING not enabled for the VPN zone (affects pinging the Sophos itself only).
3. **Return route:** servers or other firewalls lack a route to `10.81.234.0/24` via the Sophos. Add a route or enable masquerade on the rule.
4. **ICMP blocked:** Windows Firewall drops ping from other subnets. Test with a TCP port (e.g. RDP 3389).
5. **Subnet overlap:** the client's home network uses the same range as the office LAN, so traffic stays local. Test from mobile data and use a unique lease range.
6. **Policy:** with split tunnel, the internal subnets must be listed under permitted network resources.
7. **DNS:** name resolution fails while IP access works. Set DNS in the global settings.

### 7.3 Packet-level diagnosis

Run a continuous ping from the VPN client, then:

- **Log viewer > Firewall:** traffic dropped means a rule problem.
- **Diagnostics > Packet capture** (filter on the server IP): request leaves the LAN port but no reply returns, meaning a return route or host firewall problem.
- Nothing in the logs means traffic is not entering the tunnel (client routing, overlap, or policy).

### 7.4 Other log locations

- **Log viewer > SSL VPN / Authentication** for user and policy errors.
- Client-side logs for certificate or authentication failures.

---

## 8. Security Recommendations

- Enable **MFA** for VPN users.
- Use a **trusted certificate** instead of the default self-signed one.
- Restrict firewall rules to the required networks and services instead of Any/Any.
- Do not expose the User Portal or WebAdmin on WAN unless necessary.
- Review VPN logins regularly and disable unused accounts.
- Keep SFOS firmware up to date.

---

## 9. Summary of Root Causes Found in This Deployment

1. **Override hostname was blank**, so clients were pointed at a private WAN IP. Fixed by setting the DDNS hostname and re-downloading the client config.
2. **DDNS needed to be set in the router's DDNS/network settings** for the hostname to resolve and connect correctly from outside.
3. After connecting, reachability depends on the VPN-to-LAN firewall rule, return routing, and ICMP being allowed on targets (see 7.2).

---

## 10. Quick Checklist

- [ ] Router WAN IP equals the public static IP
- [ ] Port forward TCP 8443 to the Sophos WAN IP (reserved or static)
- [ ] DDNS hostname resolves to the public IP
- [ ] Device access: SSL VPN on WAN, PING on VPN
- [ ] Override hostname set to the DDNS name
- [ ] Lease range does not overlap any network
- [ ] DNS servers set
- [ ] SSL VPN policy with users and tunnel mode
- [ ] Firewall rules VPN to LAN (and VPN to WAN for full tunnel)
- [ ] Return routes or masquerade in place
- [ ] Fresh client config downloaded and tested from an external network
