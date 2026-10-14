# E. Firewall

> **Status:** 📝 Not yet tested in lab
>
> **Cert:** MTCNA module 6 (Firewall) · MTCSE

## Concept

The RouterOS firewall is **stateful** and filters packets in **chains**:

| Chain | Traffic | Example |
|---|---|---|
| `input` | **To** the router | WinBox, SSH, DNS to the router, ping to the router |
| `forward` | **Through** the router | LAN PC → internet |
| `output` | **From** the router | The router doing NTP or DNS lookups |

Rules are checked **top to bottom; the first match wins**. If nothing matches, the packet is **accepted**, so every good firewall ends with drop rules.

### Connection states

| State | Meaning |
|---|---|
| `new` | First packet of a connection |
| `established` | Part of a known connection |
| `related` | Linked to a known connection (FTP data, ICMP errors) |
| `invalid` | Broken or unknown. Drop it. |
| `untracked` | Skipped tracking (set in RAW) |

### Packet flow (simplified)

```
in → RAW prerouting → conntrack → mangle prerouting → dst-NAT → routing decision
   ├─ to router:  mangle input → FILTER INPUT → router process
   └─ through:    mangle forward → FILTER FORWARD → mangle postrouting → src-NAT → out
router process → mangle output → FILTER OUTPUT → src-NAT → out
```

### Where things are in WinBox

The firewall lives under **IP → Firewall**, with these tabs: **Filter Rules**, **NAT**, **Mangle**, **Raw**, **Service Ports**, **Connections**, **Address Lists**, **Layer7 Protocols**.

Each rule dialog has these tabs:

| Tab | What you set there |
|---|---|
| **General** | Chain, Src/Dst Address, Protocol, Src/Dst Port, In/Out Interface, Connection State, Connection NAT State |
| **Advanced** | Src/Dst Address List, TLS Host, IPsec Policy |
| **Extra** | Connection Limit, Rate Limit, Time |
| **Action** | Action, Address List + Timeout, Log + Log Prefix |

Interface **lists** are at the bottom of the General tab (In. Interface List / Out. Interface List). Rule order is changed by **drag and drop** in the list.

---

## 1. Interface lists (do this first)

🖱️ **WinBox:**
1. Interfaces → **Interface List** tab → **Lists** button → **+** → `WAN` → OK. Then **+** → `LAN` → OK.
2. Back in the **Interface List** tab → **+**:
   - List `WAN`, Interface `ether1` → OK.
   - List `LAN`, Interface `bridge1` → OK.

⌨️ **CLI:**
```
/interface list add name=WAN
/interface list add name=LAN
/interface list member add list=WAN interface=ether1
/interface list member add list=LAN interface=bridge1
```

---

## 2. Recommended base firewall (RouterOS 7)

Turn on **Safe Mode** before applying this.

### Input chain: protect the router

🖱️ **WinBox:** IP → Firewall → **Filter Rules** → **+** for each row, in this order:

| # | General tab | Action tab | Comment |
|---|---|---|---|
| 1 | Chain `input`, Connection State: `established, related, untracked` | accept | IN: accept est/rel/untracked |
| 2 | Chain `input`, Connection State: `invalid` | drop | IN: drop invalid |
| 3 | Chain `input`, Protocol `icmp` | accept | IN: accept ICMP |
| 4 | Chain `input`, Dst. Address `127.0.0.1` | accept | IN: accept loopback (CAPsMAN) |
| 5 | Chain `input`, In. Interface List: **!** `LAN` (tick the "!" box) | drop | IN: drop all not from LAN |

⌨️ **CLI:**
```
/ip firewall filter
add chain=input action=accept connection-state=established,related,untracked comment="IN: accept est/rel/untracked"
add chain=input action=drop connection-state=invalid comment="IN: drop invalid"
add chain=input action=accept protocol=icmp comment="IN: accept ICMP"
add chain=input action=accept dst-address=127.0.0.1 comment="IN: accept loopback (CAPsMAN)"
add chain=input action=drop in-interface-list=!LAN comment="IN: drop all not from LAN"
```

### Forward chain: protect the LAN

🖱️ **WinBox:** IP → Firewall → **Filter Rules** → **+**:

| # | General tab | Action tab | Comment |
|---|---|---|---|
| 6 | Chain `forward`, Connection State: `established, related` | fasttrack connection (HW Offload ✓) | FWD: fasttrack |
| 7 | Chain `forward`, Connection State: `established, related, untracked` | accept | FWD: accept est/rel/untracked |
| 8 | Chain `forward`, Connection State: `invalid` | drop | FWD: drop invalid |
| 9 | Chain `forward`, Connection State `new`, Connection NAT State **!** `dstnat`, In. Interface List `WAN` | drop | FWD: drop WAN not dstnat-ed |

⌨️ **CLI:**
```
/ip firewall filter
add chain=forward action=fasttrack-connection connection-state=established,related hw-offload=yes comment="FWD: fasttrack"
add chain=forward action=accept connection-state=established,related,untracked comment="FWD: accept est/rel/untracked"
add chain=forward action=drop connection-state=invalid comment="FWD: drop invalid"
add chain=forward action=drop connection-state=new connection-nat-state=!dstnat in-interface-list=WAN comment="FWD: drop WAN not dstnat-ed"
```

What this gives you:
- LAN → internet: ✅
- Internet → LAN: ❌, except port forwards
- Internet → router: ❌
- LAN → router: ✅

> **Fasttrack** makes established traffic skip the firewall, mangle and **queues**. Disable it (select the rule, click **✕**) when doing QoS. See [K. QoS](../K-qos/).

---

## 3. Allow management from a trusted WAN IP

### Step 1: trusted address list

🖱️ **WinBox:** IP → Firewall → **Address Lists** → **+** → Name: `MGMT-ALLOWED`, Address: `192.168.1.108` → OK

⌨️ **CLI:**
```
/ip firewall address-list add list=MGMT-ALLOWED address=192.168.1.108 comment="My PC"
```

### Step 2: accept rule placed above the drop

🖱️ **WinBox:** Filter Rules → **+**:
- General tab: Chain `input`, Protocol `tcp`, Dst. Port `8291,22`.
- Advanced tab: Src. Address List `MGMT-ALLOWED`.
- Action tab: `accept`.

Click OK, then **drag** the rule above "IN: drop all not from LAN".

⌨️ **CLI:**
```
/ip firewall filter add chain=input action=accept protocol=tcp dst-port=8291,22 src-address-list=MGMT-ALLOWED comment="IN: WinBox+SSH from trusted" place-before=[find comment="IN: drop all not from LAN"]
```

---

## 4. Address lists

🖱️ **WinBox:** IP → Firewall → **Address Lists** → **+**:
- Name: the list name, e.g. `BLOCKED`.
- Address: an IP, a subnet, or a **domain name** (resolved and updated automatically).
- Timeout: optional; the entry is auto-removed when it expires.

⌨️ **CLI:**
```
/ip firewall address-list add list=BLOCKED address=198.51.100.0/24
/ip firewall address-list add list=BLOCKED address=example-bad.com
/ip firewall address-list add list=TEMP address=1.2.3.4 timeout=1h
/ip firewall address-list print where list=BLOCKED
```

---

## 5. Brute-force protection (SSH)

Staged lists. An IP that opens too many new connections climbs the stages and ends up blacklisted for a day.

🖱️ **WinBox:** Filter Rules → **+** for each row, top to bottom, all placed **above** the input accept/drop rules. Every row uses Chain `input`, Protocol `tcp`, Dst. Port `22`.

| # | Match | Action tab |
|---|---|---|
| 1 | Advanced → Src. Address List `ssh_blacklist` | drop |
| 2 | Connection State `new` + Src. Address List `ssh_stage3` | add src to address list `ssh_blacklist`, Timeout `1d` |
| 3 | Connection State `new` + Src. Address List `ssh_stage2` | add src to address list `ssh_stage3`, Timeout `1m` |
| 4 | Connection State `new` + Src. Address List `ssh_stage1` | add src to address list `ssh_stage2`, Timeout `1m` |
| 5 | Connection State `new` | add src to address list `ssh_stage1`, Timeout `1m` |

⌨️ **CLI:**
```
/ip firewall filter
add chain=input protocol=tcp dst-port=22 src-address-list=ssh_blacklist action=drop comment="SSH: drop blacklisted"
add chain=input protocol=tcp dst-port=22 connection-state=new src-address-list=ssh_stage3 action=add-src-to-address-list address-list=ssh_blacklist address-list-timeout=1d
add chain=input protocol=tcp dst-port=22 connection-state=new src-address-list=ssh_stage2 action=add-src-to-address-list address-list=ssh_stage3 address-list-timeout=1m
add chain=input protocol=tcp dst-port=22 connection-state=new src-address-list=ssh_stage1 action=add-src-to-address-list address-list=ssh_stage2 address-list-timeout=1m
add chain=input protocol=tcp dst-port=22 connection-state=new action=add-src-to-address-list address-list=ssh_stage1 address-list-timeout=1m
```

---

## 6. Block LAN access to specific sites

🖱️ **WinBox:**
1. Address Lists → **+** → Name `BLOCK-SITES`, Address `facebook.com` → OK.
2. Filter Rules → **+**:
   - General: Chain `forward`, Src. Address `192.168.10.0/24`.
   - Advanced: Dst. Address List `BLOCK-SITES`.
   - Action: `reject`, Reject With `icmp network unreachable`.
   - Click OK, then drag the rule to the **top**.

⌨️ **CLI:**
```
/ip firewall address-list add list=BLOCK-SITES address=facebook.com
/ip firewall filter add chain=forward src-address=192.168.10.0/24 dst-address-list=BLOCK-SITES action=reject reject-with=icmp-network-unreachable place-before=0
```

The Advanced tab's **TLS Host** field (`*facebook.com`) matches HTTPS by SNI, which is more precise. It doesn't work with fasttracked connections.

---

## 7. RAW: drop early (cheap)

🖱️ **WinBox:** IP → Firewall → **Raw** → **+** → Chain `prerouting`, Advanced → Src. Address List `BLOCKED`, Action `drop` → OK

⌨️ **CLI:**
```
/ip firewall raw add chain=prerouting src-address-list=BLOCKED action=drop
```

---

## 8. Logging

🖱️ **WinBox:** Filter Rules → **+**:
- General: Chain `input`, In. Interface List `WAN`.
- Action: `log`, Log Prefix `IN-DROP: `.

Drag it above the drop rule. View the results in the **Log** window.

⌨️ **CLI:**
```
/ip firewall filter add chain=input action=log log-prefix="IN-DROP: " in-interface-list=WAN place-before=[find comment="IN: drop all not from LAN"]
/log print where message~"IN-DROP"
```

---

## 9. IPv6

IPv6 has its **own** firewall. Secure it the same way or disable IPv6.

🖱️ **WinBox:** IPv6 → Settings → Disable IPv6: ✓ (or configure IPv6 → Firewall)

⌨️ **CLI:**
```
/ipv6 settings set disable-ipv6=yes
```

---

## EVE-NG lab

```
[Cloud0 = "Internet"] --ether1(WAN)-- R1 --ether2 (bridge1, LAN)-- PC1
```

1. Create interface lists, then apply the base firewall with Safe Mode on.
2. From your Windows PC (WAN side), WinBox to `192.168.1.133` is **blocked**.
3. Add your PC to `MGMT-ALLOWED` and add the accept rule. WinBox **works**.
4. After NAT ([F](../F-nat/)), PC1 pings 8.8.8.8.
5. Add `8.8.8.8` to `BLOCK-SITES`. The PC1 ping fails.
6. Watch the counters move.

## Verify

| Check | 🖱️ WinBox | ⌨️ CLI |
|---|---|---|
| Rules + counters | IP → Firewall → Filter Rules (Bytes/Packets columns) | `/ip firewall filter print stats` |
| Live connections | IP → Firewall → **Connections** | `/ip firewall connection print` |
| Address lists | IP → Firewall → Address Lists | `/ip firewall address-list print` |
| Live traffic | Tools → **Torch** | `/tool torch interface=ether1` |
| Reset counters | Filter Rules → **Reset Counters** | `/ip firewall filter reset-counters-all` |

## Troubleshooting

| Problem | Fix |
|---|---|
| Locked out | Drop placed before your accept. Use Safe Mode, the console or MAC WinBox. |
| Rule counters stay 0 | A rule above matches first, or the wrong chain (input vs forward). |
| Port forward doesn't work | Missing dst-nat rule ([F](../F-nat/)). |
| Queues or mangle ignored | Fasttrack. Disable that rule. |
| WAN side can use your DNS | The base input chain isn't applied. |
