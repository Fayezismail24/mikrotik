# G. Static Routing

> **Status:** 📝 Not yet tested in lab

## Concept

RouterOS picks a route by:
1. **Longest prefix match:** `/24` beats `/16`, which beats `/0`.
2. Then lowest **distance**.
3. Only **active** routes (gateway reachable) are used.

| Route source | Default distance |
|---|---|
| Connected | 0 |
| Static | 1 |
| eBGP | 20 |
| OSPF | 110 |
| iBGP | 200 |

### Route flags

| Flag | Meaning |
|---|---|
| `A` | Active, being used |
| `D` | Dynamic |
| `c` | Connected |
| `s` | Static |
| `o` | OSPF |
| `b` | BGP |
| `d` | DHCP |
| `+` | ECMP |

🖱️ **In WinBox,** routes are at **IP → Routes**. The route dialog's **General** tab has Dst. Address, Gateway, Check Gateway, Distance, Scope, Target Scope and Routing Table.

---

## 1. Basic static routes

### Default route

🖱️ **WinBox:** IP → Routes → **+** → Dst. Address `0.0.0.0/0`, Gateway `192.168.1.1` → Comment `Default route` → OK

⌨️ **CLI:**
```
/ip route add dst-address=0.0.0.0/0 gateway=192.168.1.1 comment="Default route"
```

### Route to a remote LAN

🖱️ **WinBox:** IP → Routes → **+** → Dst. Address `10.2.0.0/24`, Gateway `10.0.12.2` → OK

⌨️ **CLI:**
```
/ip route add dst-address=10.2.0.0/24 gateway=10.0.12.2 comment="To R2 LAN"
```

The gateway can also be an interface (`ether2`, `wg1`, `pppoe-out1`), which is useful for tunnels.

### Blackhole

🖱️ **WinBox:** IP → Routes → **+** → Dst. Address `10.99.0.0/16`, Type `blackhole` → OK

⌨️ **CLI:**
```
/ip route add dst-address=10.99.0.0/16 blackhole
```

---

## 2. Dual-WAN failover (distance + check-gateway)

🖱️ **WinBox:** IP → Routes → **+** (two routes):

| Dst. Address | Gateway | Check Gateway | Distance | Comment |
|---|---|---|---|---|
| `0.0.0.0/0` | `192.168.1.1` | `ping` | 1 | ISP1 primary |
| `0.0.0.0/0` | `192.168.2.1` | (none) | 2 | ISP2 backup |

⌨️ **CLI:**
```
/ip route add dst-address=0.0.0.0/0 gateway=192.168.1.1 distance=1 check-gateway=ping comment="ISP1 primary"
/ip route add dst-address=0.0.0.0/0 gateway=192.168.2.1 distance=2 comment="ISP2 backup"
```

**Limitation:** this only detects a dead **gateway**, not a dead internet behind a working modem.

---

## 3. Recursive failover (detects an upstream outage)

🖱️ **WinBox:** IP → Routes → **+** (four routes):

| Dst. Address | Gateway | Check Gateway | Distance | Scope | Target Scope | Comment |
|---|---|---|---|---|---|---|
| `8.8.8.8/32` | `192.168.1.1` | none | 1 | **10** | 10 | Monitor via ISP1 |
| `1.1.1.1/32` | `192.168.2.1` | none | 1 | **10** | 10 | Monitor via ISP2 |
| `0.0.0.0/0` | `8.8.8.8` | ping | 1 | 30 | **11** | Default via ISP1 |
| `0.0.0.0/0` | `1.1.1.1` | ping | 2 | 30 | **11** | Default via ISP2 |

⌨️ **CLI:**
```
/ip route add dst-address=8.8.8.8/32 gateway=192.168.1.1 scope=10 comment="Monitor via ISP1"
/ip route add dst-address=1.1.1.1/32 gateway=192.168.2.1 scope=10 comment="Monitor via ISP2"
/ip route add dst-address=0.0.0.0/0 gateway=8.8.8.8 target-scope=11 check-gateway=ping distance=1 comment="Default via ISP1"
/ip route add dst-address=0.0.0.0/0 gateway=1.1.1.1 target-scope=11 check-gateway=ping distance=2 comment="Default via ISP2"
```

If 8.8.8.8 stops answering through ISP1, the traffic moves to ISP2. The side effect is that traffic to 8.8.8.8 itself always uses ISP1.

---

## 4. Load balancing (ECMP)

🖱️ **WinBox:** IP → Routes → **+** → Dst. Address `0.0.0.0/0`, Gateway: `192.168.1.1`, then click the **▼** arrow to add a second gateway `192.168.2.1` → OK

⌨️ **CLI:**
```
/ip route add dst-address=0.0.0.0/0 gateway=192.168.1.1,192.168.2.1
```

---

## 5. Policy-based routing (routing tables)

Send VLAN 20 out ISP2.

### Step 1: routing table

🖱️ **WinBox:** Routing → **Tables** → **+** → Name `via-isp2`, FIB ✓ → OK

⌨️ **CLI:**
```
/routing table add name=via-isp2 fib
```

### Step 2: default route in that table

🖱️ **WinBox:** IP → Routes → **+** → Dst. Address `0.0.0.0/0`, Gateway `192.168.2.1`, Routing Table `via-isp2` → OK

⌨️ **CLI:**
```
/ip route add dst-address=0.0.0.0/0 gateway=192.168.2.1 routing-table=via-isp2
```

### Step 3a: routing rule (by source)

🖱️ **WinBox:** Routing → **Rules** → **+** → Src. Address `192.168.20.0/24`, Action `lookup only in table`, Table `via-isp2` → OK

⌨️ **CLI:**
```
/routing rule add src-address=192.168.20.0/24 action=lookup-only-in-table table=via-isp2
```

### Step 3b: or mangle (more flexible, e.g. by port)

🖱️ **WinBox:** IP → Firewall → **Mangle** → **+**:
- General: Chain `prerouting`, Protocol `tcp`, Dst. Port `443`, In. Interface List `LAN`.
- Extra: Dst. Address Type `!local`.
- Action: `mark routing`, New Routing Mark `via-isp2`, Passthrough unticked.

⌨️ **CLI:**
```
/ip firewall mangle add chain=prerouting in-interface-list=LAN dst-address-type=!local protocol=tcp dst-port=443 action=mark-routing new-routing-mark=via-isp2 passthrough=no
```

Masquerade must exist for **both** WAN interfaces.

---

## 6. Route to a LAN behind another router

```
R1 (LAN 10.1.0.0/24) --10.0.12.0/30-- R2 (LAN 10.2.0.0/24)
```

### R1

🖱️ **WinBox:**
1. IP → Addresses → **+** → `10.0.12.1/30` on `ether3`.
2. IP → Routes → **+** → Dst. Address `10.2.0.0/24`, Gateway `10.0.12.2`.

⌨️ **CLI:**
```
/ip address add address=10.0.12.1/30 interface=ether3
/ip route add dst-address=10.2.0.0/24 gateway=10.0.12.2
```

### R2

🖱️ **WinBox:**
1. IP → Addresses → **+** → `10.0.12.2/30` on `ether1`.
2. IP → Routes → **+** → Dst. Address `10.1.0.0/24`, Gateway `10.0.12.1`.
3. IP → Routes → **+** → Dst. Address `0.0.0.0/0`, Gateway `10.0.12.1`.

⌨️ **CLI:**
```
/ip address add address=10.0.12.2/30 interface=ether1
/ip route add dst-address=10.1.0.0/24 gateway=10.0.12.1
/ip route add dst-address=0.0.0.0/0 gateway=10.0.12.1
```

Routes must exist **in both directions**.

---

## EVE-NG lab

```
[Cloud0] --ether1-- R1 --ether2-- PC1 (10.1.0.0/24)
                    |
                 ether3  10.0.12.0/30
                    |
                 ether1
                    R2 --ether2-- PC2 (10.2.0.0/24)
```

1. Address everything, with DHCP on both LANs.
2. Add the routes from section 6.
3. PC1 ↔ PC2 ping works.
4. Masquerade on R1 (`out-interface-list=WAN`) covers both LANs.
5. PC2 pings 8.8.8.8.
6. Remove one route, and the ping fails. Use `trace` in VPCS to see where it stops.

## Verify

| Check | 🖱️ WinBox | ⌨️ CLI |
|---|---|---|
| Routing table | IP → Routes | `/ip route print` |
| Active only | IP → Routes → filter (top-right) `active` | `/ip route print where active` |
| Which route is used | (CLI only) | `/ip route check 8.8.8.8` |
| Path | Tools → Traceroute | `/tool traceroute 10.2.0.10` |
| Ping from a source | Tools → Ping → Src. Address | `/ping 10.2.0.10 src-address=10.1.0.1` |

## Troubleshooting

| Problem | Fix |
|---|---|
| Route not active | Gateway unreachable, or check-gateway failed. |
| One-way traffic | Missing return route. |
| Failover never switches | No check-gateway, or the modem is up while the internet is down. Use recursive routing. |
| Recursive route inactive | Host route `scope=10`, default `target-scope=11`. |
| PBR not working | Fasttrack bypasses mangle. Check the mangle counters. |
