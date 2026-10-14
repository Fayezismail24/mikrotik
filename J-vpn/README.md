# J. VPN: WireGuard, IPsec, L2TP, SSTP

> **Status:** 📝 Not yet tested in lab

## Concept

| VPN | Use case | Pros | Cons |
|---|---|---|---|
| **WireGuard** | Site-to-site, road warrior | Fast, simple, modern | Key-based only, no username/password |
| **IPsec IKEv2** | Site-to-site with any vendor | Industry standard | Complex |
| **L2TP/IPsec** | Road warrior with built-in OS clients | Native on Windows, macOS, iOS, Android | Older; NAT issues |
| **SSTP** | Road warrior through strict firewalls | TCP 443 passes almost anywhere | Slow; Windows-centric |

**Recommendation:** WireGuard, unless you need vendor interop (IPsec) or clients with no app (L2TP/IPsec).

🖱️ **Where things are in WinBox:**

| What | Menu |
|---|---|
| WireGuard | **WireGuard** (WireGuard + Peers tabs) |
| IPsec | **IP → IPsec**: Profiles, Proposals, Peers, Identities, Policies, Active Peers, Installed SAs |
| L2TP / SSTP | **PPP** → Interface tab (**L2TP Server**, **SSTP Server** buttons), Profiles, Secrets, Active Connections |
| Certificates | **System → Certificates** |

## Lab topology

```
LAN-A 192.168.10.0/24 -- R1 ether1 (192.168.1.133) ---- [home LAN = "internet"] ---- (192.168.1.134) ether1 R2 -- LAN-B 192.168.20.0/24
```

---

## 1. WireGuard site-to-site

### Step 1: interface + tunnel IP (both routers)

🖱️ **WinBox:**
1. **WireGuard** → **WireGuard** tab → **+** → Name `wg-s2s`, Listen Port `13231` → OK.
2. Double-click it and copy the **Public Key**.
3. IP → Addresses → **+** → `10.255.0.1/30` (R1) or `10.255.0.2/30` (R2) on `wg-s2s`.

⌨️ **CLI:**
```
/interface wireguard add name=wg-s2s listen-port=13231
/interface wireguard print
/ip address add address=10.255.0.1/30 interface=wg-s2s     # R2: 10.255.0.2/30
```

### Step 2: peer (each side gets the **other** side's public key)

🖱️ **WinBox:** WireGuard → **Peers** tab → **+**:

| Field | R1 | R2 |
|---|---|---|
| Interface | `wg-s2s` | `wg-s2s` |
| Public Key | R2's key | R1's key |
| Endpoint | `192.168.1.134` | `192.168.1.133` |
| Endpoint Port | `13231` | `13231` |
| Allowed Address | `10.255.0.2/32, 192.168.20.0/24` | `10.255.0.1/32, 192.168.10.0/24` |
| Persistent Keepalive | `25s` | `25s` |

⌨️ **CLI (R1):**
```
/interface wireguard peers add interface=wg-s2s public-key="<R2-public-key>" endpoint-address=192.168.1.134 endpoint-port=13231 allowed-address=10.255.0.2/32,192.168.20.0/24 persistent-keepalive=25s
```

⌨️ **CLI (R2):**
```
/interface wireguard peers add interface=wg-s2s public-key="<R1-public-key>" endpoint-address=192.168.1.133 endpoint-port=13231 allowed-address=10.255.0.1/32,192.168.10.0/24 persistent-keepalive=25s
```

### Step 3: route to the remote LAN

🖱️ **WinBox:** IP → Routes → **+** → Dst. Address `192.168.20.0/24` (R1) or `192.168.10.0/24` (R2), Gateway `wg-s2s` → OK

⌨️ **CLI:**
```
/ip route add dst-address=192.168.20.0/24 gateway=wg-s2s     # R1
/ip route add dst-address=192.168.10.0/24 gateway=wg-s2s     # R2
```

### Step 4: firewall (both)

🖱️ **WinBox:** IP → Firewall → Filter Rules → **+** → Chain `input`, Protocol `udp`, Dst. Port `13231`, Action `accept`. Drag it above the drop rule.

⌨️ **CLI:**
```
/ip firewall filter add chain=input protocol=udp dst-port=13231 action=accept comment="WireGuard" place-before=[find comment="IN: drop all not from LAN"]
```

---

## 2. WireGuard road warrior (phone or laptop)

🖱️ **WinBox (R1):**
1. WireGuard → **+** → `wg-rw`, Listen Port `13232`.
2. IP → Addresses → `10.66.0.1/24` on `wg-rw`.
3. WireGuard → Peers → **+**:
   - Interface `wg-rw`
   - Public Key: the client's key
   - Allowed Address `10.66.0.2/32`
   - Comment `Fayez phone`
4. Firewall: accept UDP 13232 on input, above the drop rule.
5. Interfaces → Interface List → **+** → List `LAN`, Interface `wg-rw`.

⌨️ **CLI:**
```
/interface wireguard add name=wg-rw listen-port=13232
/ip address add address=10.66.0.1/24 interface=wg-rw
/interface wireguard peers add interface=wg-rw public-key="<phone-public-key>" allowed-address=10.66.0.2/32 comment="Fayez phone"
/ip firewall filter add chain=input protocol=udp dst-port=13232 action=accept place-before=[find comment="IN: drop all not from LAN"]
/interface list member add list=LAN interface=wg-rw
```

Client config (WireGuard app):

```ini
[Interface]
PrivateKey = <phone-private-key>
Address = 10.66.0.2/32
DNS = 192.168.10.1

[Peer]
PublicKey = <R1-wg-rw-public-key>
Endpoint = <R1-public-IP-or-DDNS>:13232
AllowedIPs = 192.168.10.0/24, 10.66.0.0/24
PersistentKeepalive = 25
```

RouterOS 7.11+ can print a ready-made client config. Run this in New Terminal:
```
/interface wireguard peers show-client-config [find comment="Fayez phone"]
```

---

## 3. IPsec IKEv2 site-to-site (PSK)

### Step 1: profile (phase 1)

🖱️ **WinBox:** IP → IPsec → **Profiles** → **+** → Name `s2s-prof`, Hash `sha256`, Encryption `aes-256`, DH Group `modp2048` → OK

⌨️ **CLI:**
```
/ip ipsec profile add name=s2s-prof hash-algorithm=sha256 enc-algorithm=aes-256 dh-group=modp2048
```

### Step 2: proposal (phase 2)

🖱️ **WinBox:** IP → IPsec → **Proposals** → **+** → Name `s2s-prop`, Auth `sha256`, Encr `aes-256 cbc`, PFS Group `modp2048` → OK

⌨️ **CLI:**
```
/ip ipsec proposal add name=s2s-prop auth-algorithms=sha256 enc-algorithms=aes-256-cbc pfs-group=modp2048
```

### Step 3: peer

🖱️ **WinBox:** IP → IPsec → **Peers** → **+** → Name `to-R2`, Address `192.168.1.134/32`, Profile `s2s-prof`, Exchange Mode `IKE2` → OK

⌨️ **CLI:**
```
/ip ipsec peer add name=to-R2 address=192.168.1.134/32 exchange-mode=ike2 profile=s2s-prof
```

### Step 4: identity

🖱️ **WinBox:** IP → IPsec → **Identities** → **+** → Peer `to-R2`, Auth. Method `pre shared key`, Secret `IpsecPSK!2026` → OK

⌨️ **CLI:**
```
/ip ipsec identity add peer=to-R2 auth-method=pre-shared-key secret="IpsecPSK!2026"
```

### Step 5: policy

🖱️ **WinBox:** IP → IPsec → **Policies** → **+**:
- General tab: Peer `to-R2`, Tunnel ✓, Src. Address `192.168.10.0/24`, Dst. Address `192.168.20.0/24`.
- Action tab: Action `encrypt`, Proposal `s2s-prop`.

⌨️ **CLI:**
```
/ip ipsec policy add src-address=192.168.10.0/24 dst-address=192.168.20.0/24 tunnel=yes peer=to-R2 proposal=s2s-prop action=encrypt
```

### Step 6: NAT bypass (above masquerade)

🖱️ **WinBox:** IP → Firewall → NAT → **+** → Chain `srcnat`, Src. Address `192.168.10.0/24`, Dst. Address `192.168.20.0/24`, Action `accept`. Drag it to the top.

⌨️ **CLI:**
```
/ip firewall nat add chain=srcnat src-address=192.168.10.0/24 dst-address=192.168.20.0/24 action=accept place-before=0 comment="No NAT to R2 LAN"
```

**R2** is the mirror image: swap the addresses and subnets, and name the peer `to-R1`.

### Step 7: firewall (both)

🖱️ **WinBox:** Filter Rules → **+**, two rules above the drop:
- Chain `input`, Protocol `udp`, Dst. Port `500,4500`, Action `accept`.
- Chain `input`, Protocol `ipsec-esp`, Action `accept`.

⌨️ **CLI:**
```
/ip firewall filter add chain=input protocol=udp dst-port=500,4500 action=accept place-before=[find comment="IN: drop all not from LAN"]
/ip firewall filter add chain=input protocol=ipsec-esp action=accept place-before=[find comment="IN: drop all not from LAN"]
```

### Step 8: keep fasttrack away from IPsec

🖱️ **WinBox:** Filter Rules → double-click the **fasttrack** rule → **Advanced** tab → IPsec Policy `out, none` → OK

⌨️ **CLI:**
```
/ip firewall filter set [find action=fasttrack-connection] ipsec-policy=out,none
```

### Verify IPsec

| Check | 🖱️ WinBox | ⌨️ CLI |
|---|---|---|
| Phase 1 up | IP → IPsec → **Active Peers** | `/ip ipsec active-peers print` |
| Phase 2 SAs | IP → IPsec → **Installed SAs** | `/ip ipsec installed-sa print` |
| Policy counters | IP → IPsec → Policies | `/ip ipsec policy print stats` |

---

## 4. L2TP/IPsec road warrior

### Step 1: IP pool

🖱️ **WinBox:** IP → Pool → **+** → `vpn-pool`, `10.77.0.10-10.77.0.50`

⌨️ **CLI:**
```
/ip pool add name=vpn-pool ranges=10.77.0.10-10.77.0.50
```

### Step 2: PPP profile

🖱️ **WinBox:** PPP → **Profiles** → **+**:
- Name `l2tp-prof`
- Local Address `10.77.0.1`
- Remote Address `vpn-pool`
- DNS Server `192.168.10.1`
- Protocols tab: Use Encryption `required`

⌨️ **CLI:**
```
/ppp profile add name=l2tp-prof local-address=10.77.0.1 remote-address=vpn-pool dns-server=192.168.10.1 use-encryption=required
```

### Step 3: L2TP server

🖱️ **WinBox:** PPP → Interface tab → **L2TP Server** button:
- Enabled ✓
- Default Profile `l2tp-prof`
- Authentication: `mschap2` only
- Use IPsec `required`
- IPsec Secret `L2tpIpsecKey!`

⌨️ **CLI:**
```
/interface l2tp-server server set enabled=yes default-profile=l2tp-prof use-ipsec=required ipsec-secret="L2tpIpsecKey!" authentication=mschap2
```

### Step 4: user

🖱️ **WinBox:** PPP → **Secrets** → **+** → Name `fayez`, Password, Service `l2tp`, Profile `l2tp-prof`

⌨️ **CLI:**
```
/ppp secret add name=fayez password="UserPass!" service=l2tp profile=l2tp-prof
```

### Step 5: firewall

🖱️ **WinBox:** Filter Rules → accept on input, above the drop:
- UDP `500,1701,4500`.
- Protocol `ipsec-esp`.

⌨️ **CLI:**
```
/ip firewall filter add chain=input protocol=udp dst-port=500,1701,4500 action=accept place-before=[find comment="IN: drop all not from LAN"]
/ip firewall filter add chain=input protocol=ipsec-esp action=accept place-before=[find comment="IN: drop all not from LAN"]
```

**Windows client:** Settings → VPN → Add. Type: **L2TP/IPsec with pre-shared key**.

---

## 5. SSTP road warrior (TCP 443)

### Step 1: certificates

🖱️ **WinBox:** System → **Certificates** → **+**:
1. Name `ca`, Common Name `MyCA`, Key Usage tab: `key cert. sign`, `crl sign` → **Apply** → **Sign**.
2. **+** → Name `sstp-server`, Common Name: your public IP or DNS name → **Apply** → **Sign** with CA `ca`.
3. Select `ca` → **Export**. Download the `.crt` from Files and import it into the client's **Trusted Root** store.

⌨️ **CLI:**
```
/certificate add name=ca common-name=MyCA key-usage=key-cert-sign,crl-sign
/certificate sign ca
/certificate add name=sstp-server common-name=<public-IP-or-DNS>
/certificate sign sstp-server ca=ca
/certificate export-certificate ca
```

### Step 2: server and user

🖱️ **WinBox:**
1. PPP → Interface → **SSTP Server**:
   - Enabled ✓
   - Certificate `sstp-server`
   - Default Profile `l2tp-prof`
   - Authentication `mschap2`
2. PPP → Secrets → **+** → Service `sstp`.
3. Firewall: accept TCP 443 on input.

⌨️ **CLI:**
```
/interface sstp-server server set enabled=yes certificate=sstp-server default-profile=l2tp-prof authentication=mschap2
/ppp secret add name=fayez-sstp password="UserPass!" service=sstp profile=l2tp-prof
/ip firewall filter add chain=input protocol=tcp dst-port=443 action=accept place-before=[find comment="IN: drop all not from LAN"]
```

---

## EVE-NG lab

1. Build R1 and R2 with LANs plus VPCS. Both WANs on Cloud0.
2. Base firewall and NAT on both.
3. **WireGuard s2s:** PC-A pings PC-B.
4. Replace it with **IPsec IKEv2**, and check Installed SAs.
5. **Road warrior:** install the WireGuard app on Windows, connect to R1 `192.168.1.133:13232`, and ping PC-A.

## Verify (all VPNs)

| Check | 🖱️ WinBox | ⌨️ CLI |
|---|---|---|
| WireGuard handshake | WireGuard → Peers → **Last Handshake** column | `/interface wireguard peers print detail` |
| PPP users connected | PPP → **Active Connections** | `/ppp active print` |
| VPN logs | **Log** window (filter `ipsec` / `l2tp`) | `/log print where topics~"ipsec\|l2tp"` |

## Troubleshooting

| Problem | Fix |
|---|---|
| No WireGuard handshake | Wrong public key (use the **other side's**), UDP blocked, or wrong endpoint. |
| Handshake OK, no traffic | `allowed-address` missing the remote LAN, or no route via the WG interface. |
| IPsec phase 1 fails | PSK, profile or address mismatch. Check the log. |
| IPsec SA up, no traffic | Missing NAT bypass, or fasttrack. |
| L2TP connects, no LAN access | Forward chain drops it. Add `all-ppp` to the LAN list. |
| SSTP certificate error | The client doesn't trust the CA, or the CN doesn't match the address used. |
