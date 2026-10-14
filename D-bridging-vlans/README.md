# D. Bridging and VLANs

> **Status:** 📝 Not yet tested in lab
>
> **Cert:** MTCNA module 3 (Bridging) · MTCRE (VLANs)

## Concept

- **Bridge:** a software switch. Ports added to a bridge share one L2 network, and the IP goes on the **bridge**, not on the ports.
- **VLAN:** splits one physical network into separate L2 networks with an 802.1Q tag (ID 1–4094).
  - **Access port:** one VLAN, untagged. PCs and printers connect here.
  - **Trunk port:** many VLANs, tagged. It links to switches, APs and routers.
  - **PVID:** the VLAN that untagged frames arriving on a port get assigned to.
- **Bridge VLAN filtering:** the modern RouterOS way. One bridge handles all VLANs and can be hardware-offloaded.
- **Router-on-a-stick:** a VLAN interface per VLAN on top of the bridge, each with a gateway IP. The router routes between VLANs.

> Avoid old methods: one bridge per VLAN, or VLAN interfaces on bridged ports. They break hardware offload and cause confusing bugs.

---

## 1. Simple bridge (no VLANs)

### Create the bridge

🖱️ **WinBox:** Bridge → **Bridge** tab → **+** → Name: `bridge-lan` → OK

⌨️ **CLI:**
```
/interface bridge add name=bridge-lan
```

### Add ports

🖱️ **WinBox:** Bridge → **Ports** tab → **+** → Interface: `ether2`, Bridge: `bridge-lan` → OK. Repeat for ether3 and ether4.

⌨️ **CLI:**
```
/interface bridge port add bridge=bridge-lan interface=ether2
/interface bridge port add bridge=bridge-lan interface=ether3
/interface bridge port add bridge=bridge-lan interface=ether4
```

### IP on the bridge

🖱️ **WinBox:** IP → Addresses → **+** → Address: `192.168.10.1/24`, Interface: `bridge-lan` → OK

⌨️ **CLI:**
```
/ip address add address=192.168.10.1/24 interface=bridge-lan
```

If a DHCP server was on `ether2`, move it to the bridge: IP → DHCP Server → double-click it → Interface: `bridge-lan`.

### Useful bridge views and settings

| Task | 🖱️ WinBox | ⌨️ CLI |
|---|---|---|
| MAC table | Bridge → **Hosts** tab | `/interface bridge host print` |
| STP mode | Bridge → Bridge → double-click → **STP** tab → Protocol Mode: `RSTP` | `/interface bridge set bridge-lan protocol-mode=rstp` |
| Hardware offload | Bridge → Ports (H flag / HW column) | `/interface bridge port print` |
| IGMP snooping | Bridge → Bridge → double-click → General → IGMP Snooping ✓ | `/interface bridge set bridge-lan igmp-snooping=yes` |

---

## 2. VLAN design for this lab

| VLAN | Name | Subnet | Gateway |
|---|---|---|---|
| 10 | USERS | 192.168.10.0/24 | 192.168.10.1 |
| 20 | VOIP | 192.168.20.0/24 | 192.168.20.1 |
| 99 | MGMT | 192.168.99.0/24 | 192.168.99.1 |

| R1 port | Role |
|---|---|
| ether2 | Access, VLAN 10 |
| ether3 | Access, VLAN 20 |
| ether4 | Trunk: tagged 10, 20, 99 |

---

## 3. Bridge VLAN filtering config

**Order matters. Turn on VLAN filtering last.** Use Safe Mode.

### Step 1: Bridge with filtering off

🖱️ **WinBox:** Bridge → **Bridge** tab → **+** → Name: `bridge1` → **VLAN** tab → VLAN Filtering: **unticked** → OK

⌨️ **CLI:**
```
/interface bridge add name=bridge1 vlan-filtering=no
```

### Step 2: Ports (access ports get a PVID)

🖱️ **WinBox:** Bridge → **Ports** tab → **+**:
- ether2: Bridge `bridge1`. On the **VLAN** tab: PVID `10`, Frame Types `admit only untagged and priority tagged`.
- ether3: Bridge `bridge1`. On the **VLAN** tab: PVID `20`, same Frame Types.
- ether4: Bridge `bridge1`. On the **VLAN** tab: PVID `1`, Frame Types `admit only VLAN tagged`.

⌨️ **CLI:**
```
/interface bridge port add bridge=bridge1 interface=ether2 pvid=10 frame-types=admit-only-untagged-and-priority-tagged
/interface bridge port add bridge=bridge1 interface=ether3 pvid=20 frame-types=admit-only-untagged-and-priority-tagged
/interface bridge port add bridge=bridge1 interface=ether4 frame-types=admit-only-vlan-tagged
```

### Step 3: VLAN table (bridge1 must be tagged so the router can route each VLAN)

🖱️ **WinBox:** Bridge → **VLANs** tab → **+**:

| Bridge | VLAN IDs | Tagged | Untagged |
|---|---|---|---|
| bridge1 | 10 | bridge1, ether4 | ether2 |
| bridge1 | 20 | bridge1, ether4 | ether3 |
| bridge1 | 99 | bridge1, ether4 | (none) |

⌨️ **CLI:**
```
/interface bridge vlan add bridge=bridge1 vlan-ids=10 tagged=bridge1,ether4 untagged=ether2
/interface bridge vlan add bridge=bridge1 vlan-ids=20 tagged=bridge1,ether4 untagged=ether3
/interface bridge vlan add bridge=bridge1 vlan-ids=99 tagged=bridge1,ether4
```

### Step 4: VLAN interfaces (L3)

🖱️ **WinBox:** Interfaces → **+** → **VLAN**:
- Name `vlan10-users`, VLAN ID `10`, Interface `bridge1` → OK
- Name `vlan20-voip`, VLAN ID `20`, Interface `bridge1` → OK
- Name `vlan99-mgmt`, VLAN ID `99`, Interface `bridge1` → OK

⌨️ **CLI:**
```
/interface vlan add interface=bridge1 name=vlan10-users vlan-id=10
/interface vlan add interface=bridge1 name=vlan20-voip  vlan-id=20
/interface vlan add interface=bridge1 name=vlan99-mgmt  vlan-id=99
```

### Step 5: Gateway IPs

🖱️ **WinBox:** IP → Addresses → **+** (once per VLAN):
- `192.168.10.1/24` on `vlan10-users`
- `192.168.20.1/24` on `vlan20-voip`
- `192.168.99.1/24` on `vlan99-mgmt`

⌨️ **CLI:**
```
/ip address add address=192.168.10.1/24 interface=vlan10-users
/ip address add address=192.168.20.1/24 interface=vlan20-voip
/ip address add address=192.168.99.1/24 interface=vlan99-mgmt
```

### Step 6: Turn on filtering

🖱️ **WinBox:** Bridge → Bridge → double-click `bridge1` → **VLAN** tab → VLAN Filtering: ✓ → OK

⌨️ **CLI:**
```
/interface bridge set bridge1 vlan-filtering=yes
```

### DHCP per VLAN

🖱️ **WinBox:** easiest is IP → DHCP Server → **DHCP Setup**, run once for `vlan10-users` and once for `vlan20-voip`.

⌨️ **CLI:**
```
/ip pool add name=pool10 ranges=192.168.10.100-192.168.10.200
/ip pool add name=pool20 ranges=192.168.20.100-192.168.20.200
/ip dhcp-server network add address=192.168.10.0/24 gateway=192.168.10.1 dns-server=192.168.10.1
/ip dhcp-server network add address=192.168.20.0/24 gateway=192.168.20.1 dns-server=192.168.20.1
/ip dhcp-server add name=dhcp10 interface=vlan10-users address-pool=pool10 disabled=no
/ip dhcp-server add name=dhcp20 interface=vlan20-voip  address-pool=pool20 disabled=no
```

### Block routing between VLANs

🖱️ **WinBox:**
1. Interfaces → **Interface List** tab → **Lists** → **+** → Name: `VLANS` → OK.
2. Back in the **Interface List** tab → **+**:
   - List `VLANS`, Interface `vlan10-users` → OK.
   - Again for `vlan20-voip`.
3. IP → Firewall → **Filter Rules** → **+**:
   - General tab: Chain `forward`, In. Interface List `VLANS`, Out. Interface List `VLANS`.
   - Action tab: `drop`.
   - Click OK, then drag the rule below the established/related accept rule.

⌨️ **CLI:**
```
/interface list add name=VLANS
/interface list member add list=VLANS interface=vlan10-users
/interface list member add list=VLANS interface=vlan20-voip
/ip firewall filter add chain=forward in-interface-list=VLANS out-interface-list=VLANS action=drop comment="Block inter-VLAN"
```

---

## 4. Trunk to a second MikroTik (switch role)

SW1: ether1 is the trunk from R1, ether2 is an access port for VLAN 10.

🖱️ **WinBox (on SW1):**
1. Bridge → **+** `bridge1` (VLAN Filtering off).
2. Bridge → **Ports**:
   - ether1: Frame Types `admit only VLAN tagged`.
   - ether2: PVID `10`, Frame Types `admit only untagged and priority tagged`.
3. Bridge → **VLANs**:
   - VLAN 10: Tagged `ether1`, Untagged `ether2`.
   - VLAN 99: Tagged `bridge1, ether1`.
4. Interfaces → **+ VLAN**: `vlan99-mgmt`, ID 99 on `bridge1`.
5. IP → Addresses: `192.168.99.2/24` on `vlan99-mgmt`.
6. IP → Routes → **+**: Gateway `192.168.99.1`.
7. Bridge → `bridge1` → VLAN Filtering ✓.

⌨️ **CLI:**
```
/interface bridge add name=bridge1 vlan-filtering=no
/interface bridge port add bridge=bridge1 interface=ether1 frame-types=admit-only-vlan-tagged
/interface bridge port add bridge=bridge1 interface=ether2 pvid=10 frame-types=admit-only-untagged-and-priority-tagged
/interface bridge vlan add bridge=bridge1 vlan-ids=10 tagged=ether1 untagged=ether2
/interface bridge vlan add bridge=bridge1 vlan-ids=99 tagged=bridge1,ether1
/interface vlan add interface=bridge1 name=vlan99-mgmt vlan-id=99
/ip address add address=192.168.99.2/24 interface=vlan99-mgmt
/ip route add gateway=192.168.99.1
/interface bridge set bridge1 vlan-filtering=yes
```

---

## EVE-NG lab

```
[Cloud0] --ether1-- R1 --ether2 (VLAN10 access)-- PC1
                       --ether3 (VLAN20 access)-- PC2
                       --ether4 (trunk)-- SW1 (CHR) --ether2 (VLAN10 access)-- PC3
```

1. Build R1 (section 3) with DHCP per VLAN.
2. Build SW1 (section 4).
3. PC1 and PC3 (both VLAN 10) get `192.168.10.x` and ping each other through the trunk.
4. PC2 gets `192.168.20.x`.
5. PC1 → PC2 ping works (routed). Add the isolation rule, and it fails.
6. From R1, ping SW1 at `192.168.99.2`.

## Verify

| Check | 🖱️ WinBox | ⌨️ CLI |
|---|---|---|
| VLAN table | Bridge → VLANs (current tagged/untagged) | `/interface bridge vlan print` |
| PVIDs | Bridge → Ports | `/interface bridge port print` |
| MACs per VLAN | Bridge → Hosts | `/interface bridge host print` |
| VLAN interfaces | Interfaces → VLAN tab | `/interface vlan print` |

## Troubleshooting

| Problem | Fix |
|---|---|
| Lost access after enabling VLAN filtering | The management IP isn't on a VLAN in the table, or `bridge1` wasn't tagged. Use Safe Mode, the console or MAC WinBox. |
| VLAN interface gets no traffic | `bridge1` missing from Tagged for that VLAN. |
| Access port client gets nothing | Wrong PVID, or the port is missing from Untagged. |
| Trunk doesn't pass a VLAN | The VLAN isn't tagged on the trunk port on **both** sides. |
| DHCP not answering | Bind it to the **VLAN interface**, not the bridge or port. |
