# C. Addressing: IP, DHCP, DNS

> **Status:** 📝 Not yet tested in lab

## Concept

- **IP address:** assigned to an interface with a prefix, e.g. `192.168.10.1/24`. Adding one automatically creates a **connected route**.
- **DHCP client:** gets an IP, gateway and DNS from upstream (ISP or home router).
- **DHCP server:** hands out IPs to LAN clients. It needs three parts: an **IP pool**, a **network** (gateway/DNS to give out), and a **server** bound to an interface.
- **DNS:** the router resolves names for itself and, with `allow-remote-requests=yes`, works as a DNS cache for the LAN.

### Quick subnet reference

| Prefix | Mask | Usable hosts |
|---|---|---|
| /30 | 255.255.255.252 | 2 (point-to-point links) |
| /29 | 255.255.255.248 | 6 |
| /28 | 255.255.255.240 | 14 |
| /27 | 255.255.255.224 | 30 |
| /26 | 255.255.255.192 | 62 |
| /25 | 255.255.255.128 | 126 |
| /24 | 255.255.255.0 | 254 |
| /32 | 255.255.255.255 | 1 (loopbacks, host routes) |

---

## 1. Static IP addresses

### Add

🖱️ **WinBox:** IP → Addresses → **+**:
- Address: `192.168.10.1/24`
- Interface: `ether2`
- Comment (yellow note): `LAN`

Click **OK**.

⌨️ **CLI:**
```
/ip address add address=192.168.10.1/24 interface=ether2 comment="LAN"
```

### View and remove

🖱️ **WinBox:** IP → Addresses → select the row → **−** (remove), or **✕** (disable)

⌨️ **CLI:**
```
/ip address print
/ip address remove [find address="192.168.10.1/24"]
```

### Loopback address

🖱️ **WinBox:** IP → Addresses → **+** → Address: `10.255.255.1/32`, Interface: `lo` → OK

⌨️ **CLI:**
```
/ip address add address=10.255.255.1/32 interface=lo
```

---

## 2. DHCP client (WAN side)

### Add

🖱️ **WinBox:** IP → DHCP Client → **+**:
- Interface: `ether1`
- Use Peer DNS: ✓
- Use Peer NTP: ✓
- Add Default Route: `yes`

Click **OK**. The Status column should show `bound`.

⌨️ **CLI:**
```
/ip dhcp-client add interface=ether1 add-default-route=yes use-peer-dns=yes use-peer-ntp=yes disabled=no
/ip dhcp-client print detail
```

### Renew or release

🖱️ **WinBox:** IP → DHCP Client → double-click the entry → **Renew** / **Release**

⌨️ **CLI:**
```
/ip dhcp-client renew [find interface=ether1]
/ip dhcp-client release [find interface=ether1]
```

---

## 3. DHCP server (LAN side)

### Option A: wizard

🖱️ **WinBox:** IP → DHCP Server → **DHCP Setup** button → follow the prompts: interface, network, gateway, pool, DNS, lease time.

⌨️ **CLI:**
```
/ip dhcp-server setup
```

### Option B: manual (know what the wizard builds)

**Step 1: pool**

🖱️ **WinBox:** IP → Pool → **+** → Name: `pool-lan`, Addresses: `192.168.10.100-192.168.10.200` → OK

⌨️ **CLI:**
```
/ip pool add name=pool-lan ranges=192.168.10.100-192.168.10.200
```

**Step 2: network**

🖱️ **WinBox:** IP → DHCP Server → **Networks** tab → **+**:
- Address: `192.168.10.0/24`
- Gateway: `192.168.10.1`
- DNS Servers: `192.168.10.1`
- Domain: `lab.local`

Click **OK**.

⌨️ **CLI:**
```
/ip dhcp-server network add address=192.168.10.0/24 gateway=192.168.10.1 dns-server=192.168.10.1 domain=lab.local
```

**Step 3: server**

🖱️ **WinBox:** IP → DHCP Server → **DHCP** tab → **+**:
- Name: `dhcp-lan`
- Interface: `ether2`
- Address Pool: `pool-lan`
- Lease Time: `1d`

Click **OK**, then make sure it's enabled (not grey).

⌨️ **CLI:**
```
/ip dhcp-server add name=dhcp-lan interface=ether2 address-pool=pool-lan lease-time=1d disabled=no
```

### Leases and reservations

🖱️ **WinBox:**
- IP → DHCP Server → **Leases** tab shows connected clients.
- To reserve one: select a lease → **Make Static**.
- To add one manually: **+** → Address, MAC Address, Server → OK.

⌨️ **CLI:**
```
/ip dhcp-server lease print
/ip dhcp-server lease make-static [find address=192.168.10.150]
/ip dhcp-server lease add address=192.168.10.20 mac-address=AA:BB:CC:DD:EE:FF server=dhcp-lan comment="Printer"
```

### DHCP options (example: option 66, TFTP server for VoIP phones)

🖱️ **WinBox:**
1. IP → DHCP Server → **Options** tab → **+** → Name: `tftp`, Code: `66`, Value: `'192.168.10.5'` → OK.
2. Then **Networks** tab → double-click the network → DHCP Options: `tftp` → OK.

⌨️ **CLI:**
```
/ip dhcp-server option add name=tftp code=66 value="'192.168.10.5'"
/ip dhcp-server network set [find address=192.168.10.0/24] dhcp-option=tftp
```

String values need the inner single quotes.

---

## 4. DNS

### Settings

🖱️ **WinBox:** IP → DNS:
- Servers: `1.1.1.1`, `8.8.8.8`
- Allow Remote Requests: ✓
- Cache Size: `4096` KiB

Click **Apply**.

⌨️ **CLI:**
```
/ip dns set servers=1.1.1.1,8.8.8.8 allow-remote-requests=yes cache-size=4096KiB
/ip dns print
```

> **Security:** `allow-remote-requests=yes` makes the router answer DNS on **every** interface, WAN included. Always block port 53 from WAN in the firewall (see [E. Firewall](../E-firewall/)).

### Cache

🖱️ **WinBox:** IP → DNS → **Cache** button. **Flush Cache** clears it.

⌨️ **CLI:**
```
/ip dns cache print
/ip dns cache flush
```

### Static entries

🖱️ **WinBox:** IP → DNS → **Static** button → **+** → Name: `router.lab.local`, Address: `192.168.10.1` → OK

⌨️ **CLI:**
```
/ip dns static add name=router.lab.local address=192.168.10.1
/ip dns static add name=nas.lab.local address=192.168.10.20
```

### DNS over HTTPS (DoH)

🖱️ **WinBox:** IP → DNS → Use DoH Server: `https://cloudflare-dns.com/dns-query`, Verify DoH Certificate: ✓ → Apply

⌨️ **CLI:**
```
/ip dns set use-doh-server=https://cloudflare-dns.com/dns-query verify-doh-cert=yes
```

Verifying the certificate needs a CA bundle in System → Certificates. For a lab, you can untick it.

---

## 5. ARP

### View and add static ARP

🖱️ **WinBox:** IP → ARP → the list shows current entries. **+** → Address, MAC Address, Interface → OK adds a static one.

⌨️ **CLI:**
```
/ip arp print
/ip arp add address=192.168.10.20 mac-address=AA:BB:CC:DD:EE:FF interface=ether2
```

### Only allow DHCP clients (reply-only ARP)

🖱️ **WinBox:**
1. IP → DHCP Server → **DHCP** tab → double-click `dhcp-lan` → Add ARP For Leases: ✓.
2. Interfaces → double-click `ether2` → ARP: `reply-only`.

⌨️ **CLI:**
```
/ip dhcp-server set dhcp-lan add-arp=yes
/interface ethernet set ether2 arp=reply-only
```

---

## EVE-NG lab

```
[Cloud0/home LAN] --ether1-- R1 --ether2-- [VPCS PC1]
```

1. R1 ether1: DHCP client (it already has one).
2. R1 ether2: `192.168.10.1/24`.
3. Build the DHCP server manually (Option B).
4. DNS: `servers=1.1.1.1`, `allow-remote-requests=yes`.
5. On PC1 (VPCS): `ip dhcp`, then `show ip`. It should get `192.168.10.100` or similar.
6. Make PC1's lease static.
7. Add a static DNS entry `r1.lab.local` and test it with `/ping r1.lab.local` on R1.

PC1 reaches R1 but **not the internet** yet. That needs NAT, covered in [F. NAT](../F-nat/).

## Verify

| Check | 🖱️ WinBox | ⌨️ CLI |
|---|---|---|
| IPs | IP → Addresses | `/ip address print` |
| Connected routes | IP → Routes (DAc) | `/ip route print` |
| DHCP server | IP → DHCP Server → DHCP | `/ip dhcp-server print` |
| Leases | IP → DHCP Server → Leases | `/ip dhcp-server lease print` |
| DNS | IP → DNS | `/ip dns print` |
| Ping / resolve | Tools → Ping | `/ping 192.168.10.100` · `/resolve google.com` |

## Troubleshooting

| Problem | Fix |
|---|---|
| Client gets no lease | Server on the wrong interface (on a bridge, bind it to the **bridge**), server disabled, or pool exhausted. |
| IP but no internet | Missing NAT masquerade or default route. |
| Clients can't resolve names | `allow-remote-requests=no`, or DNS Servers missing in the DHCP Network. |
| Two DHCP servers on the LAN | Rogue device. Detect it with IP → DHCP Server → **Alerts**. |
| `/resolve` fails on the router | No DNS servers, or no route out. |
