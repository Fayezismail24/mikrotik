# G. Static Routing

> **Status:** 📝 Not yet tested in lab
>
> **Cert:** MTCNA module 4 (Routing) · MTCRE (static routing, /32 addressing, VRRP)

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

### Scope and target-scope explained (MTCRE)

**The problem they solve:** a route's gateway must itself be reachable through **another** route. RouterOS needs rules for which routes may be used to "find" a gateway. Without them, routes could resolve through each other in loops.

- **scope:** "how far up this route can help." It's this route's rank when **other** routes look for their gateway through it.
- **target-scope:** "how far down I'm allowed to look." When resolving **this** route's gateway, RouterOS only uses routes whose **scope ≤ my target-scope**.

**Analogy:** asking for directions. `target-scope` is how far you're willing to trust someone, and `scope` is how trusted each person is. You only accept directions from someone whose trust level is within your limit.

| Route | Default scope | Default target-scope | Meaning |
|---|---|---|---|
| Connected (`DAc`) | 10 | n/a | Directly attached; the most trusted |
| Static | 30 | **10** | Its gateway must be on a **connected** network (scope 10) |
| OSPF | 20 | 10 | |
| iBGP | 40 | 30 | Next hop can be found through static or IGP routes |

**Why the recursive example works:**
1. The default route's gateway is `8.8.8.8`, which is **not** on a connected network.
2. With the default `target-scope=10`, RouterOS would only look at scope-10 routes. The host route `8.8.8.8/32` has default scope 30, so it **wouldn't** be used and the default route would stay inactive.
3. So we set the host route to `scope=10` and the default route to `target-scope=11`. Now `10 ≤ 11`, so the host route is allowed to resolve `8.8.8.8`. The default route goes **recursively** through `8.8.8.8/32` to `192.168.1.1`.

Check it with `/ip route print detail`. The active default shows `immediate-gw`, the real next hop it resolved to.

---

## 3b. /32 point-to-point addressing (MTCRE)

### What is it?

Normally a link between two routers uses a **/30** subnet: 4 addresses, of which only 2 are usable. **/32 addressing** gives each router **one** address and tells it directly which single address is on the other side.

```
/30 way:  R1 10.0.12.1/30 ---- R2 10.0.12.2/30   (uses 10.0.12.0–3, 4 addresses)
/32 way:  R1 10.0.0.1/32  ---- R2 10.0.0.2/32    (uses only the 2 addresses)
```

### Why use it

- Saves addresses: important with public IPs, where every address costs money.
- Common with ISPs, PPP links and some MTCRE scenarios.

### /30 vs /32

| | /30 | /32 with `network=` |
|---|---|---|
| Addresses used | 4 | 2 |
| Config | Simple, works everywhere | Must set `network=` (the peer IP) |
| Neighbor in the same subnet? | Yes | No, so the **network** field tells the router where the peer is |

### Config

🖱️ **WinBox:** IP → Addresses → **+**:

| Router | Address | Network | Interface |
|---|---|---|---|
| R1 | `10.0.0.1` | `10.0.0.2` | `ether3` |
| R2 | `10.0.0.2` | `10.0.0.1` | `ether1` |

⌨️ **CLI:**
```
# R1
/ip address add address=10.0.0.1/32 network=10.0.0.2 interface=ether3
# R2
/ip address add address=10.0.0.2/32 network=10.0.0.1 interface=ether1
```

RouterOS creates a connected route to the peer address (`10.0.0.2/32` on R1), so you can use it as a gateway: `/ip route add dst-address=10.2.0.0/24 gateway=10.0.0.2`.

**Verify:** 🖱️ IP → Routes shows `10.0.0.2` as **DAc**. ⌨️ `/ping 10.0.0.2`

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

## 7. VRRP: a backup gateway (MTCRE)

### What is it?

Every PC on a LAN has **one** default gateway, say `192.168.10.1`. If that router dies, the whole LAN loses the internet, even if you have a second router.

**VRRP (Virtual Router Redundancy Protocol)** lets two or more routers **share one virtual gateway IP**. One router is the **Master** and answers for that IP; the others are **Backup**. If the master dies, a backup takes over the IP within a few seconds. The PCs never notice, because their gateway IP never changes.

**Analogy:** a shop with one public phone number. Normally the manager answers. If the manager is out, the call automatically rings the assistant. Customers always dial the same number.

### Key terms

| Term | Meaning |
|---|---|
| **VRID** | Virtual router ID (1–255). It must be the **same** on all routers in the group. |
| **Virtual IP** | The shared gateway IP that PCs use |
| **Priority** | 1–254, default 100. The **highest** priority becomes master. |
| **Preemption** | When the original master comes back, it takes over again (on by default) |

### VRRP vs two gateways vs routing protocols

| | Two routers, no VRRP | VRRP | OSPF / BGP |
|---|---|---|---|
| Protects | Nothing: PCs still point to one gateway | The **LAN gateway** | **Paths between routers** |
| PC config changes on failure | Manual | None | n/a (PCs don't run OSPF) |
| Typical use | n/a | Two edge routers serving one LAN | Router-to-router redundancy |

### Lab design

```
                 (internet / Cloud0)
                  |               |
                 R1               R2
            ether2 .2         ether2 .3
                  \               /
                 LAN 192.168.10.0/24 (switch)
                         |
                        PC1  gateway = 192.168.10.1 (virtual)
```

| | R1 (master) | R2 (backup) |
|---|---|---|
| Real LAN IP | `192.168.10.2/24` | `192.168.10.3/24` |
| VRID | 10 | 10 |
| Priority | **200** | 100 |
| Virtual IP | `192.168.10.1` | `192.168.10.1` |

### Step 1: real IPs (each router)

🖱️ **WinBox:** IP → Addresses → **+** → `192.168.10.2/24` (R1) or `192.168.10.3/24` (R2) on `ether2`

⌨️ **CLI:**
```
/ip address add address=192.168.10.2/24 interface=ether2     # R2: 192.168.10.3/24
```

### Step 2: VRRP interface

🖱️ **WinBox:** Interfaces → **+** → **VRRP**:
- Name `vrrp10`
- Interface `ether2`
- VRID `10`
- Priority `200` (R1) or `100` (R2)

⌨️ **CLI:**
```
/interface vrrp add name=vrrp10 interface=ether2 vrid=10 priority=200    # R2: priority=100
```

### Step 3: virtual IP on the VRRP interface (as /32)

🖱️ **WinBox:** IP → Addresses → **+** → Address `192.168.10.1/32`, Interface `vrrp10` (same on both routers)

⌨️ **CLI:**
```
/ip address add address=192.168.10.1/32 interface=vrrp10
```

### Step 4: firewall

VRRP routers send "I'm alive" adverts to each other. Allow them on input:

🖱️ **WinBox:** Filter Rules → **+** → Chain `input`, Protocol `vrrp`, Action `accept`. Drag it above the drop rule.

⌨️ **CLI:**
```
/ip firewall filter add chain=input protocol=vrrp action=accept place-before=[find comment="IN: drop all not from LAN"]
```

### Step 5: DHCP and NAT

Give clients the **virtual IP** as their gateway: `gateway=192.168.10.1` in the DHCP network. Run DHCP on only **one** router, or split the pool between both. Each router needs its own masquerade.

### Verify and test failover

| Check | 🖱️ WinBox | ⌨️ CLI |
|---|---|---|
| Who is master | Interfaces → `vrrp10` (**M** flag / running on master only) | `/interface vrrp print` |
| Virtual IP active | IP → Addresses (active only on master) | `/ip address print` |

Test:
1. Run a continuous ping from PC1 to 8.8.8.8.
2. Disable R1's `ether2`, or shut R1 down.
3. R2 becomes master and the ping recovers after a few lost packets.
4. Re-enable R1. With preemption, it takes the master role back.

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
