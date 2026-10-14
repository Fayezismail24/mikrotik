# H. Dynamic Routing: OSPF and BGP

> **Status:** 📝 Not yet tested in lab
>
> **Cert:** MTCRE (OSPF) · 🔹 BGP is beyond MTCRE (MTCINE)

## Concept

Dynamic routing protocols let routers **share routes automatically** and react to failures.

| | OSPF | BGP |
|---|---|---|
| Type | Link-state IGP | Path-vector EGP |
| Used for | Inside one organization | Between ISPs and organizations, or large internal networks |
| Picks paths by | Lowest cost | Policy (AS path, local-pref, MED...) |
| Distance | 110 | 20 (eBGP) / 200 (iBGP) |

> RouterOS 7 rewrote OSPF and BGP. The v6 commands and menus don't exist anymore. Everything here is v7.

🖱️ **In WinBox:**
- **Routing → OSPF** has the tabs Instances, Areas, Interface Templates, Interfaces, Neighbors, LSAs.
- **Routing → BGP** has the tabs Connections, Templates, Sessions, Advertisements.
- **Routing → Filters** holds the routing filter rules.

## Lab topology

```
             10.0.12.0/30            10.0.23.0/30
  R1 ether2 ------------ ether1 R2 ether2 ------------ ether1 R3
  lo 1.1.1.1/32                 lo 2.2.2.2/32                 lo 3.3.3.3/32
  ether3: 10.1.0.0/24 LAN       ether3: 10.2.0.0/24 LAN       ether3: 10.3.0.0/24 LAN
  ether1: Cloud0 (internet)

  Optional: R1 ether4 -- 10.0.13.0/30 -- R3 ether4 (redundant path)
```

### Base addressing (R1 shown; same pattern on R2 and R3)

🖱️ **WinBox:** IP → Addresses → **+**:
- `1.1.1.1/32` on `lo`
- `10.0.12.1/30` on `ether2`
- `10.1.0.1/24` on `ether3`

⌨️ **CLI:**
```
/ip address add address=1.1.1.1/32 interface=lo
/ip address add address=10.0.12.1/30 interface=ether2
/ip address add address=10.1.0.1/24 interface=ether3
```

---

## Part 1: OSPF

Key ideas:
- **Router ID:** unique; use the loopback IP.
- **Area 0 (backbone):** every area must connect to it.
- **Passive interface:** advertised, but no hellos sent. Use it on LANs.
- **Network type `ptp`:** for /30 links. It's faster and skips the DR/BDR election.

### Step 1: instance

🖱️ **WinBox:** Routing → OSPF → **Instances** → **+**:
- Name: `ospf1`
- Version: `2`
- Router ID: `1.1.1.1`
- Redistribute: `connected` (optional)
- Originate Default: `if-installed` (on R1 only, which has the internet)

⌨️ **CLI:**
```
/routing ospf instance add name=ospf1 version=2 router-id=1.1.1.1 redistribute=connected originate-default=if-installed
```

On R2 and R3: `router-id=2.2.2.2` / `3.3.3.3`, with no `originate-default`.

### Step 2: area

🖱️ **WinBox:** Routing → OSPF → **Areas** → **+** → Name `backbone`, Area ID `0.0.0.0`, Instance `ospf1` → OK

⌨️ **CLI:**
```
/routing ospf area add name=backbone area-id=0.0.0.0 instance=ospf1
```

### Step 3: interface templates

🖱️ **WinBox:** Routing → OSPF → **Interface Templates** → **+**:
- Template 1: Interfaces `ether2`, Area `backbone`, Type `ptp` → OK.
- Template 2: Interfaces `ether3`, `lo`, Area `backbone`, Passive ✓ → OK.

⌨️ **CLI (R1):**
```
/routing ospf interface-template add area=backbone interfaces=ether2 type=ptp
/routing ospf interface-template add area=backbone interfaces=ether3,lo passive
```

**R2** (two transit links):
```
/routing ospf interface-template add area=backbone interfaces=ether1,ether2 type=ptp
/routing ospf interface-template add area=backbone interfaces=ether3,lo passive
```

**R3:**
```
/routing ospf interface-template add area=backbone interfaces=ether1 type=ptp
/routing ospf interface-template add area=backbone interfaces=ether3,lo passive
```

You can also match by subnet with the template's **Networks** field (`networks=10.0.0.0/8`).

### Cost (make a link the backup)

🖱️ **WinBox:** Routing → OSPF → Interface Templates → double-click the `ether4` template → Cost `100` → OK

⌨️ **CLI:**
```
/routing ospf interface-template set [find interfaces=ether4] cost=100
```

### Authentication (MD5)

🖱️ **WinBox:** Interface Templates → double-click the template → Auth `md5`, Auth ID `1`, Auth Key `OspfKey123` → OK. It must match on both sides.

⌨️ **CLI:**
```
/routing ospf interface-template set [find interfaces=ether2] auth=md5 auth-id=1 auth-key="OspfKey123"
```

### Verify OSPF

| Check | 🖱️ WinBox | ⌨️ CLI |
|---|---|---|
| Neighbors (must be **Full**) | Routing → OSPF → Neighbors | `/routing ospf neighbor print` |
| OSPF interfaces | Routing → OSPF → Interfaces | `/routing ospf interface print` |
| LSDB | Routing → OSPF → LSAs | `/routing ospf lsa print` |
| Learned routes | IP → Routes (flag `o`) | `/ip route print where ospf` |

### OSPF troubleshooting

| Problem | Fix |
|---|---|
| No neighbor | Area, subnet, network type or MTU mismatch. Firewall blocks protocol 89. Interface passive by mistake. |
| Stuck in ExStart | MTU mismatch. |
| Neighbor Full but no routes | LANs not in any template, or no redistribute. |

---

## Part 1b: OSPF in depth (MTCRE)

### Why multiple areas?

In one big area, **every** router stores **every** link of the whole network and recalculates every time any link flaps. That's fine for 5 routers, but heavy for 200.

**Areas** split the network into neighborhoods. Inside an area, routers know every detail. Between areas, they only exchange **summaries**.

**Analogy:** a country's postal system. Local post offices know every street in their city (one area). The national hub (area 0) only needs to know which city each postcode belongs to.

### Router roles

| Role | Meaning |
|---|---|
| **Internal router** | All interfaces in one area |
| **Backbone router** | Has an interface in **area 0** |
| **ABR** (Area Border Router) | Connects area 0 to another area, and summarizes between them |
| **ASBR** (AS Boundary Router) | Brings **outside** routes into OSPF (redistributing static, connected or BGP) |

**Rule:** every area must touch **area 0**. Non-backbone areas talk to each other **through** area 0.

### LSA types (what routers tell each other)

| Type | Name | Sent by | Travels | Describes |
|---|---|---|---|---|
| 1 | Router LSA | Every router | Its own area only | "My links and their costs" |
| 2 | Network LSA | The DR on a broadcast segment | Own area | "Routers on this shared segment" |
| 3 | Summary LSA | ABR | Into other areas | "Networks from another area" |
| 4 | ASBR Summary | ABR | Into other areas | "How to reach the ASBR" |
| 5 | External LSA | ASBR | Whole OSPF domain (not stub areas) | "Outside routes (static, connected, BGP)" |
| 7 | NSSA External | ASBR inside an NSSA | Only the NSSA (the ABR converts it to type 5) | "Outside routes from inside an NSSA" |

### Area types

| Area type | Gets type 3 (inter-area)? | Gets type 5 (external)? | Can have an ASBR inside? | Use |
|---|---|---|---|---|
| **Standard** | ✅ | ✅ | ✅ | Default |
| **Stub** | ✅ | ❌ Replaced by a default route | ❌ | Branches that only need "go to the ABR for outside" |
| **Totally stubby** (stub + no-summaries) | ❌ Only a default | ❌ | ❌ | Smallest routing tables |
| **NSSA** | ✅ | ❌ | ✅ (as type 7) | A stub area that still connects to an outside network |

**Why stub areas?** Fewer routes means less memory and CPU on small branch routers. The branch just needs a default route to the ABR.

### Multi-area lab

```
  Area 0 (backbone)            Area 1 (stub)
R1 ether2 ---- ether1 R2 ether2 ---- ether1 R3
10.0.12.0/30   (ABR)       10.0.23.0/30
```

R2 is the **ABR**. R3 is in a **stub** area.

**R1**: everything in area 0, as in Part 1.

**R2 (ABR): two areas**

🖱️ **WinBox:**
1. Routing → OSPF → **Areas** → **+** `backbone` (`0.0.0.0`) and **+** `area1` (`0.0.0.1`, Type `stub`).
2. **Interface Templates**:
   - `ether1` in `backbone`, type `ptp`.
   - `ether2` in `area1`, type `ptp`.

⌨️ **CLI:**
```
/routing ospf instance add name=ospf1 version=2 router-id=2.2.2.2
/routing ospf area add name=backbone area-id=0.0.0.0 instance=ospf1
/routing ospf area add name=area1 area-id=0.0.0.1 instance=ospf1 type=stub
/routing ospf interface-template add area=backbone interfaces=ether1 type=ptp
/routing ospf interface-template add area=area1 interfaces=ether2 type=ptp
/routing ospf interface-template add area=backbone interfaces=lo passive
```

**R3: in area 1 only** (the area type must match: **stub** on both R2 and R3)

🖱️ **WinBox:** Areas → **+** `area1` (`0.0.0.1`, Type `stub`). Interface Templates: `ether1` (ptp), and `ether3`, `lo` (passive), all in `area1`.

⌨️ **CLI:**
```
/routing ospf instance add name=ospf1 version=2 router-id=3.3.3.3
/routing ospf area add name=area1 area-id=0.0.0.1 instance=ospf1 type=stub
/routing ospf interface-template add area=area1 interfaces=ether1 type=ptp
/routing ospf interface-template add area=area1 interfaces=ether3,lo passive
```

**Totally stubby:** on the **ABR (R2)** only, block type 3 as well:

🖱️ **WinBox:** Areas → `area1` → No Summaries ✓

⌨️ **CLI:**
```
/routing ospf area set area1 no-summaries=yes
```

### Area ranges (summarization on the ABR)

Advertise **one** summary instead of many small networks:

🖱️ **WinBox:** Routing → OSPF → **Area Ranges** → **+** → Area `area1`, Prefix `10.3.0.0/16`, Advertise ✓

⌨️ **CLI:**
```
/routing ospf area range add area=area1 prefix=10.3.0.0/16 advertise=yes
```

### Redistribution (making a router an ASBR)

Bring non-OSPF routes (static, connected, BGP) into OSPF as **external** (type 5) routes:

🖱️ **WinBox:** Routing → OSPF → **Instances** → `ospf1` → Redistribute: `static`, `connected`

⌨️ **CLI:**
```
/routing ospf instance set ospf1 redistribute=static,connected
```

Default route options on the instance (**Originate Default**):

| Value | Meaning |
|---|---|
| `never` | Don't advertise a default route |
| `if-installed` | Advertise a default only if this router has one (e.g. from DHCP) |
| `always` | Always advertise a default, even without one (careful: black-holes traffic if there's no internet) |

### Verify multi-area

| Check | 🖱️ WinBox | ⌨️ CLI |
|---|---|---|
| LSAs by type | Routing → OSPF → LSAs (Type column) | `/routing ospf lsa print` |
| R3 only has a default + intra-area routes | IP → Routes on R3 | `/ip route print where ospf` |
| Area type | Routing → OSPF → Areas | `/routing ospf area print` |

**Expected on R3 (stub):** no type-5 external routes, just a **default route** from R2. With totally stubby, inter-area routes disappear too, leaving only the default.

### Quick check

1. Why must every area connect to area 0?
2. What's the difference between an ABR and an ASBR?
3. Which LSA type disappears in a stub area, and what replaces it?

---

## Part 2: BGP

Key ideas:
- **AS number:** use private ASNs 64512–65534 for labs.
- **eBGP** is between different ASes, **iBGP** within the same AS.
- No auto-discovery: each **connection** is configured manually (TCP 179).
- BGP only advertises what you tell it to.

| Router | AS |
|---|---|
| R1 | 65001 |
| R2 | 65002 |
| R3 | 65003 |

### Step 1: networks to advertise (address list)

🖱️ **WinBox:** IP → Firewall → **Address Lists** → **+** → Name `bgp-out`, Address `10.1.0.0/24` (R1) → OK

⌨️ **CLI:**
```
/ip firewall address-list add list=bgp-out address=10.1.0.0/24
```

R2 uses `10.2.0.0/24`, R3 uses `10.3.0.0/24`.

### Step 2: BGP connection

🖱️ **WinBox:** Routing → BGP → **Connections** → **+**:
- Name: `to-R2`
- AS: `65001`
- Router ID: `1.1.1.1`
- **Remote** section: Address `10.0.12.2`, AS `65002`
- **Local** section: Role `ebgp`
- **Output** tab: Network `bgp-out`

⌨️ **CLI (R1):**
```
/routing bgp connection add name=to-R2 as=65001 router-id=1.1.1.1 local.role=ebgp remote.address=10.0.12.2 remote.as=65002 output.network=bgp-out
```

**R2** (two connections):
```
/ip firewall address-list add list=bgp-out address=10.2.0.0/24
/routing bgp connection add name=to-R1 as=65002 router-id=2.2.2.2 local.role=ebgp remote.address=10.0.12.1 remote.as=65001 output.network=bgp-out
/routing bgp connection add name=to-R3 as=65002 router-id=2.2.2.2 local.role=ebgp remote.address=10.0.23.2 remote.as=65003 output.network=bgp-out
```

**R3:**
```
/ip firewall address-list add list=bgp-out address=10.3.0.0/24
/routing bgp connection add name=to-R2 as=65003 router-id=3.3.3.3 local.role=ebgp remote.address=10.0.23.1 remote.as=65002 output.network=bgp-out
```

`output.network` only advertises a prefix if a matching route exists. Connected routes count.

### Templates (shared settings)

🖱️ **WinBox:** Routing → BGP → **Templates** → **+** → Name `ebgp-default`, AS `65002`, Router ID `2.2.2.2`. Then in a connection, set Templates: `ebgp-default`.

⌨️ **CLI:**
```
/routing bgp template add name=ebgp-default as=65002 router-id=2.2.2.2
/routing bgp connection add name=to-R1 templates=ebgp-default local.role=ebgp remote.address=10.0.12.1 remote.as=65001
```

### Route filters

🖱️ **WinBox:**
1. Routing → **Filters** → **Rules** → **+**:
   - Chain: `from-R1`
   - Rule: `if (dst in 10.0.0.0/8 && dst-len == 24) { accept } else { reject }`
2. Routing → BGP → Connections → `to-R1` → **Input** tab → Filter: `from-R1`.

⌨️ **CLI:**
```
/routing filter rule add chain=from-R1 rule="if (dst in 10.0.0.0/8 && dst-len == 24) { accept } else { reject }"
/routing bgp connection set to-R1 input.filter=from-R1
```

More filter examples (same place in WinBox):
```
/routing filter rule add chain=from-ISP1 rule="set bgp-local-pref 200; accept"
/routing filter rule add chain=to-ISP2 rule="set bgp-path-prepend 3; accept"
```

### MD5 password

🖱️ **WinBox:** Routing → BGP → Connections → double-click → TCP MD5 Key `BgpKey123`

⌨️ **CLI:**
```
/routing bgp connection set to-R2 tcp-md5-key="BgpKey123"
```

### Verify BGP

| Check | 🖱️ WinBox | ⌨️ CLI |
|---|---|---|
| Connections | Routing → BGP → Connections | `/routing bgp connection print` |
| Session state (**established**) | Routing → BGP → Sessions | `/routing bgp session print` |
| What we send | Routing → BGP → Advertisements | `/routing bgp advertisements print` |
| Learned routes | IP → Routes (flag `b`) | `/ip route print where bgp` |
| Refresh after a filter change | Sessions → select → **Refresh** | `/routing bgp session refresh [find]` |

### BGP troubleshooting

| Problem | Fix |
|---|---|
| Not established | Wrong remote address or AS, no reachability, firewall blocks TCP 179 in **input**. |
| Established, nothing received | The peer's `bgp-out` list is empty, or there's no matching route on the peer. |
| Received but not active | No route to the next-hop. |

---

## Firewall note

With the base firewall from [E](../E-firewall/), allow the routing protocols in the **input** chain, above the drop rule:

🖱️ **WinBox:** IP → Firewall → Filter Rules → **+**:
- Rule 1: Chain `input`, Protocol `ospf`, Action `accept`.
- Rule 2: Chain `input`, Protocol `tcp`, Dst. Port `179`, Action `accept`.

Drag both above "IN: drop all not from LAN".

⌨️ **CLI:**
```
/ip firewall filter add chain=input protocol=ospf action=accept place-before=[find comment="IN: drop all not from LAN"]
/ip firewall filter add chain=input protocol=tcp dst-port=179 action=accept place-before=[find comment="IN: drop all not from LAN"]
```

## EVE-NG lab

1. Build R1–R3 with a VPCS on each LAN. Base addressing only.
2. **OSPF:** configure it, and the neighbors reach Full. All PCs ping each other, and R2/R3 learn a default route from R1.
3. Add the R1–R3 link with `cost=100`. Disable R1–R2 and watch the traffic reroute.
4. **BGP:** remove OSPF (Routing → OSPF → Instances → remove), configure eBGP, and all PCs ping again.
5. Add a filter on R2 that rejects `10.3.0.0/24` toward R1. R1 loses that route.
