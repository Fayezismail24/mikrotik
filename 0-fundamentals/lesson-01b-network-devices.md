# Lesson 1b: Network Devices, What Each One Does

> **Cert:** MTCNA module 1 (Introduction) and general networking · **Status:** 📝 Written, hands-on not yet done

[← Lesson 1](lesson-01-router-and-mikrotik.md) · [Fundamentals index](README.md) · [Next: Lesson 2, IP addresses →](README.md)

---

## Why this lesson?

"Router", "switch" and "access point" get used loosely. A home box is called a router but is really four devices in one. To design and troubleshoot networks, you need to know **what each device does on its own**, **which layer it works at**, and **what it reads** to make decisions.

---

## 1. The OSI layers: where each device works

Each layer adds its own information to the data. A device works at the **highest layer it reads** to make its decision.

```
 Layer   Name            What it reads / deals with            Devices working here
 ─────   ─────────────   ───────────────────────────────────   ───────────────────────────────
  7      Application     Apps: HTTP, DNS, SMTP                 Firewall (app control), proxy
  6      Presentation    Encryption, formats (TLS)             ─
  5      Session         Sessions between apps                 ─
  4      Transport       Ports: TCP / UDP                      Firewall (stateful), NAT
  3      Network         IP addresses                          Router, L3 switch, firewall
  2      Data Link       MAC addresses, VLAN tags              Switch, access point, bridge
  1      Physical        Bits, signals, cables, radio          Hub, cable, modem/ONT, repeater
```

**Analogy:** sending a parcel.
- **L1** is the truck and the road.
- **L2** is the label for the **next stop** (the MAC address, which changes at each hop).
- **L3** is the **final** destination address (the IP, which stays the same end to end).
- **L4** is **which person** in the building should receive it (the port).

---

## 2. Device by device

### Hub (legacy): Layer 1

- **What:** a box with ports that **repeats every bit to every other port**.
- **How it decides:** it doesn't. Everyone gets everything.
- **Problem:** only one device can talk at a time, so ports collide, and everyone sees everyone's traffic.
- **Analogy:** a room where everyone shouts and everyone hears every message.
- **Today:** obsolete. Learn it only to understand **why** switches exist.

### Switch: Layer 2

- **What:** connects devices **inside one network** (one subnet / one VLAN).
- **How it decides:** it learns the **source MAC** of every frame into a **MAC table** (MAC → port). It then sends each frame **only** to the port where the destination MAC lives.
- **Unknown destination or broadcast** (`FF:FF:FF:FF:FF:FF`): it **floods** the frame out of all ports in that VLAN.
- **Analogy:** a receptionist who learns which office each person sits in and delivers mail only there.
- **MikroTik:**
  - **CRS** (Cloud Router Switch) and **CSS** (Cloud Smart Switch) are dedicated switches.
  - Any RouterOS device acts as a switch when ports are in a **bridge**.

### L3 switch: Layer 2 + Layer 3

- **What:** a switch that can **also route between VLANs**, in **hardware** (the switch chip).
- **How it decides:** MAC table inside a VLAN, routing table between VLANs.
- **Strength:** very fast inter-VLAN routing.
- **Weakness:** usually limited or no NAT, VPN or advanced firewall.
- **Analogy:** a big office building with internal elevators between floors, but no exit to the street.
- **MikroTik:** **CRS3xx / CRS5xx** with **L3 hardware offload** in RouterOS 7.

### Router: Layer 3

- **What:** connects **different networks** and picks the path between them.
- **How it decides:** looks up the **destination IP** in the **routing table**.
- **Also typically does:** NAT, firewall, DHCP, DNS, VPN, PPPoE, QoS.
- **Analogy:** a post office that sends parcels between cities.
- **MikroTik:** **hEX / RB5009** (small to mid), **CCR** (Cloud Core Router, ISP-grade), **CHR** (virtual).

### Access point (AP): Layer 2, wireless

- **What:** connects **Wi-Fi devices** into a wired network. In pure AP mode it **bridges** wireless to wired and does **no** routing.
- **How it decides:** like a switch, by MAC address, but over radio.
- **Analogy:** a door that lets people walk into the building without a hallway (no cable).
- **MikroTik:** **cAP** (ceiling), **wAP** (outdoor or wall), plus any hAP with routing turned off.

### Wi-Fi router (home "router"): all in one

- **What:** **router + switch + AP + DHCP + NAT + firewall** in one box.
- **MikroTik:** **hAP** (home Access Point) and **Chateau** (with LTE/5G).

### Modem / ONT: Layer 1–2, ISP side

- **What:** converts the ISP's signal into Ethernet.
  - **Modem:** DSL (phone line), cable (coax), LTE/5G (cellular).
  - **ONT:** Optical Network Terminal, for fiber (GPON).
- It does **not** route by itself. Many ISP boxes are a **modem + router** combined.
- **Analogy:** the translator at the border who converts the outside language into yours.
- **MikroTik:** LTE modems in **LtAP** and **Chateau**, and SFP GPON modules.

### Firewall appliance: Layer 3–7

- **What:** a device whose main job is **security**: stateful filtering, and often app control, IPS and VPN.
- **How it decides:** IPs, ports, connection state, and sometimes the application or content.
- **Analogy:** the security guard at the building entrance who checks every visitor.
- **MikroTik:** RouterOS has a full stateful firewall built in ([E. Firewall](../E-firewall/)), so a MikroTik router is also your firewall.

### Wireless controller: management

- **What:** manages many APs from **one place**: same SSIDs, security and channels everywhere.
- **MikroTik:** **CAPsMAN**, built into RouterOS ([I. Wireless and CAPsMAN](../I-wireless-capsman/)).

---

## 3. Comparison tables

### Hub vs switch

| | Hub | Switch |
|---|---|---|
| Layer | 1 | 2 |
| Sends a frame to | **All** ports | **Only** the destination port (after learning) |
| Learns MACs | ❌ | ✅ MAC table |
| Collisions | Yes, all ports share one collision domain | No, each port is its own collision domain |
| Duplex | Half | Full |
| Privacy | Everyone sees everything | Only broadcasts are seen by all |
| Today | Obsolete | Everywhere |

### Switch vs L3 switch vs router

| | Switch (L2) | L3 switch | Router |
|---|---|---|---|
| Forwards by | **MAC address** | MAC **and** IP | **IP address** |
| Works within | One subnet / VLAN | Many VLANs, routing between them | Different networks, including the internet |
| How it routes | Doesn't | In **hardware** (switch chip), so very fast | In **software** (CPU), so more flexible |
| NAT, VPN, full firewall, PPPoE | ❌ | Usually ❌ or limited | ✅ |
| Typical job | Connect PCs in an office | Route between internal VLANs | Connect the site to the internet and other sites |
| MikroTik | CSS, CRS (SwOS or bridge) | CRS3xx/5xx with L3 HW offload | hEX, RB5009, CCR, CHR |

**Key idea:** an L3 switch is for **fast routing inside** the network. A router is for the **edge**: NAT, firewall, VPN, ISP links.

### Router vs firewall

| | Router | Firewall appliance |
|---|---|---|
| Main job | Move packets between networks | Decide which packets are **allowed** |
| Default stance | Usually forwards everything | Usually blocks everything not allowed |
| Deep inspection (apps, IPS) | Basic | Advanced |
| MikroTik | Both in one: RouterOS routes **and** has a full stateful firewall | |

### Access point vs Wi-Fi router

| | Access point | Wi-Fi router |
|---|---|---|
| Routes / NAT | ❌ It bridges into the existing network | ✅ |
| DHCP server | ❌ (the main router does it) | ✅ |
| Typical use | Extend Wi-Fi in a building, many APs | One box at home |
| MikroTik | cAP, wAP, or a hAP in bridge/CAP mode | hAP, Chateau |

### Modem vs router

| | Modem / ONT | Router |
|---|---|---|
| Works at | L1–L2 (signal conversion) | L3 (IP routing) |
| Connects | ISP line ↔ Ethernet | Your LAN ↔ the internet |
| NAT, DHCP, firewall | ❌ (if it's a pure modem) | ✅ |
| Common setup | ISP modem in **bridge mode** → your MikroTik does the routing | |

### Collision domain vs broadcast domain

- **Collision domain:** devices whose signals can **collide**, because they share one medium.
- **Broadcast domain:** devices that **all receive** a broadcast frame (e.g. an ARP request). This is the same as one subnet / one VLAN.

| Device | Splits collision domains? | Splits broadcast domains? |
|---|---|---|
| Hub | ❌ | ❌ |
| Switch | ✅ Every port | ❌ (one per VLAN) |
| Switch with VLANs | ✅ | ✅ One per **VLAN** |
| Router | ✅ | ✅ Every interface |

**Why it matters:** too many devices in one broadcast domain means every ARP and DHCP broadcast hits everyone, and the network slows down. That's why we split networks with **VLANs and routers** ([D](../D-bridging-vlans/), [G](../G-static-routing/)).

---

## 4. The MikroTik product family

| Family | Name means | Role | OS | Examples |
|---|---|---|---|---|
| **hAP** | home Access Point | Router + switch + Wi-Fi AP | RouterOS | hAP ax², hAP ax³, hAP ac² |
| **hEX / RB** | RouterBOARD | Wired router | RouterOS | hEX, RB5009 |
| **CCR** | Cloud Core Router | High-performance router (ISP, data center) | RouterOS | CCR2004, CCR2116 |
| **CRS** | Cloud Router Switch | Managed / L3 switch | RouterOS **or** SwOS (dual boot) | CRS326, CRS309, CRS518 |
| **CSS** | Cloud Smart Switch | Simple managed switch | **SwOS only** | CSS610, CSS326 |
| **cAP / wAP** | ceiling AP / wall-weather AP | Access points | RouterOS | cAP ax, wAP ax |
| **SXT / LHG / mANTBox** | Outdoor wireless | Point-to-point links, WISP CPE and sectors | RouterOS | LHG 5, SXT LTE |
| **LtAP / Chateau** | LTE AP | Routers with LTE/5G modems | RouterOS | Chateau 5G, LtAP mini |
| **CHR** | Cloud Hosted Router | **Virtual** router (VMs, cloud, EVE-NG) | RouterOS | Our lab R1 |

### RouterOS vs SwOS

| | RouterOS | SwOS |
|---|---|---|
| What | Full router OS | Lightweight **switch-only** OS |
| Routing, NAT, firewall, VPN | ✅ | ❌ |
| VLANs, port isolation, LAG, basic ACL | ✅ | ✅ |
| Managed with | WinBox, WebFig, CLI, API | **Web browser only** |
| Learning curve | Higher | Very simple |
| Runs on | All RouterOS devices; CRS can dual-boot | CSS, and CRS in SwOS mode |

**Rule of thumb:** a CRS used purely as a switch can run **SwOS** for simplicity. Use **RouterOS** when you need L3 features or want everything in WinBox.

---

## 5. Hands-on: one MikroTik, two roles

The same RouterOS box can act as a **switch** or as a **router**, depending on how you configure it.

### Lab topology

```
           (switch role)                         (router role)
PC1 ── ether2 ┐                             PC1 ── ether2 (bridge1) 192.168.10.1/24
              ├── bridge1 (R1)                                 │ R1 routes
PC2 ── ether3 ┘                             PC3 ── ether4       192.168.20.1/24
```

Add **three VPCS** nodes in EVE-NG: PC1 → R1 ether2, PC2 → R1 ether3, PC3 → R1 ether4. Stop R1 before adding the links.

### Part A: R1 as a switch (Layer 2)

**Step 1: bridge ether2 and ether3**

🖱️ **WinBox:**
1. Bridge → **+** → Name `bridge1` → OK.
2. Bridge → **Ports** → **+** → `ether2` on `bridge1` → OK. Repeat for `ether3`.

⌨️ **CLI:**
```
/interface bridge add name=bridge1
/interface bridge port add bridge=bridge1 interface=ether2
/interface bridge port add bridge=bridge1 interface=ether3
```

**Step 2: give the PCs addresses in the same subnet** (VPCS consoles):
```
PC1> ip 192.168.10.11/24
PC2> ip 192.168.10.12/24
PC1> ping 192.168.10.12
```

The ping works with **no IP on R1 at all**. R1 is switching purely by MAC address, like any L2 switch.

**Step 3: look at the MAC table**

🖱️ **WinBox:** Bridge → **Hosts** tab. You see PC1's and PC2's MACs and which port each was learned on.

⌨️ **CLI:**
```
/interface bridge host print
```

### Part B: R1 as a router (Layer 3)

**Step 4: give R1 a gateway IP in each network**

🖱️ **WinBox:** IP → Addresses → **+**:
- `192.168.10.1/24` on `bridge1`
- `192.168.20.1/24` on `ether4`

⌨️ **CLI:**
```
/ip address add address=192.168.10.1/24 interface=bridge1
/ip address add address=192.168.20.1/24 interface=ether4
```

**Step 5: PCs in different subnets, with R1 as the gateway**
```
PC1> ip 192.168.10.11/24 192.168.10.1
PC3> ip 192.168.20.13/24 192.168.20.1
PC1> ping 192.168.20.13
PC1> trace 192.168.20.13
```

The ping works **through R1**. `trace` shows the hop `192.168.10.1`: R1 is now **routing** between two networks.

**Step 6: look at the routing table**

🖱️ **WinBox:** IP → **Routes**. There are two **DAc** routes, one for each connected network.

⌨️ **CLI:**
```
/ip route print
```

### What you just proved

| | Part A | Part B |
|---|---|---|
| Role | Switch | Router |
| Decides by | MAC (Bridge → Hosts) | IP (IP → Routes) |
| PCs are in | **Same** subnet | **Different** subnets |
| R1 needs an IP? | ❌ | ✅ One per network (the gateway) |

---

## ✅ Quick check

1. Why can PC1 and PC2 ping each other in Part A even though R1 has no IP address?
2. A network has 300 PCs in one VLAN and is slow because of broadcasts. Which device or feature fixes that, and why?
3. You need fast routing between 10 internal VLANs, **and** NAT plus VPN to the internet. Which MikroTik devices would you put where?

---

[← Lesson 1](lesson-01-router-and-mikrotik.md) · [Fundamentals index](README.md) · [Next: Lesson 2, IP addresses →](README.md)
