# D. Bridging and VLANs

> **Status:** 📝 Not yet tested in lab

## Concept

- **Bridge:** a software switch. Ports you add to a bridge share one L2 network, and the IP goes on the **bridge**, not on the ports.
- **VLAN:** splits one physical network into separate L2 networks using an 802.1Q tag (VLAN ID 1–4094).
  - **Access port:** carries one VLAN, untagged. PCs and printers connect here.
  - **Trunk port:** carries many VLANs, tagged. It links to switches, APs and routers.
  - **PVID:** the VLAN that untagged frames arriving on a port get assigned to.
- **Bridge VLAN filtering:** the modern RouterOS way to do VLANs. One bridge handles all VLANs, and switch chips can hardware-offload it.
- **Router-on-a-stick:** the router has a VLAN interface per VLAN on top of the bridge. Each one gets a gateway IP, so the router routes between VLANs.

> Avoid old methods: one bridge per VLAN, or VLAN interfaces on ports that are also bridge members. They break hardware offload and cause confusing bugs.

## 1. Simple bridge (no VLANs)

```
/interface bridge add name=bridge-lan
/interface bridge port add bridge=bridge-lan interface=ether2
/interface bridge port add bridge=bridge-lan interface=ether3
/interface bridge port add bridge=bridge-lan interface=ether4
/ip address add address=192.168.10.1/24 interface=bridge-lan
```

If a DHCP server was on `ether2`, move it to the bridge: `/ip dhcp-server set dhcp-lan interface=bridge-lan`.

### Useful bridge features

```
/interface bridge host print                    # MAC table
/interface bridge set bridge-lan protocol-mode=rstp
/interface bridge port print                    # H flag = hardware offloaded
/interface bridge set bridge-lan igmp-snooping=yes
```

## 2. VLAN design for this lab

| VLAN | Name | Subnet | Gateway |
|---|---|---|---|
| 10 | USERS | 192.168.10.0/24 | 192.168.10.1 |
| 20 | VOIP | 192.168.20.0/24 | 192.168.20.1 |
| 99 | MGMT | 192.168.99.0/24 | 192.168.99.1 |

Ports on R1:

| Port | Role |
|---|---|
| ether2 | Access, VLAN 10 |
| ether3 | Access, VLAN 20 |
| ether4 | Trunk: tagged 10, 20, 99 (to a switch or second router) |

## 3. Bridge VLAN filtering config

**Do it in this order. Turn on `vlan-filtering=yes` last**, or you may cut yourself off. Use Safe Mode (`Ctrl+X`).

```
# 1) Bridge with filtering off for now
/interface bridge add name=bridge1 vlan-filtering=no

# 2) Ports: access ports get a PVID, trunk keeps pvid=1
/interface bridge port add bridge=bridge1 interface=ether2 pvid=10 frame-types=admit-only-untagged-and-priority-tagged
/interface bridge port add bridge=bridge1 interface=ether3 pvid=20 frame-types=admit-only-untagged-and-priority-tagged
/interface bridge port add bridge=bridge1 interface=ether4 frame-types=admit-only-vlan-tagged

# 3) VLAN table: bridge1 must be tagged so the router itself can route each VLAN
/interface bridge vlan add bridge=bridge1 vlan-ids=10 tagged=bridge1,ether4 untagged=ether2
/interface bridge vlan add bridge=bridge1 vlan-ids=20 tagged=bridge1,ether4 untagged=ether3
/interface bridge vlan add bridge=bridge1 vlan-ids=99 tagged=bridge1,ether4

# 4) VLAN interfaces (L3, router-on-a-stick) on top of the bridge
/interface vlan add interface=bridge1 name=vlan10-users vlan-id=10
/interface vlan add interface=bridge1 name=vlan20-voip  vlan-id=20
/interface vlan add interface=bridge1 name=vlan99-mgmt  vlan-id=99

# 5) Gateway IPs
/ip address add address=192.168.10.1/24 interface=vlan10-users
/ip address add address=192.168.20.1/24 interface=vlan20-voip
/ip address add address=192.168.99.1/24 interface=vlan99-mgmt

# 6) Turn on filtering
/interface bridge set bridge1 vlan-filtering=yes
```

### DHCP per VLAN

```
/ip pool add name=pool10 ranges=192.168.10.100-192.168.10.200
/ip pool add name=pool20 ranges=192.168.20.100-192.168.20.200
/ip dhcp-server network add address=192.168.10.0/24 gateway=192.168.10.1 dns-server=192.168.10.1
/ip dhcp-server network add address=192.168.20.0/24 gateway=192.168.20.1 dns-server=192.168.20.1
/ip dhcp-server add name=dhcp10 interface=vlan10-users address-pool=pool10 disabled=no
/ip dhcp-server add name=dhcp20 interface=vlan20-voip  address-pool=pool20 disabled=no
```

### Isolate VLANs from each other

By default the router routes between all VLANs. To block that:

```
/interface list add name=VLANS
/interface list member add list=VLANS interface=vlan10-users
/interface list member add list=VLANS interface=vlan20-voip
/ip firewall filter add chain=forward in-interface-list=VLANS out-interface-list=VLANS action=drop comment="Block inter-VLAN"
```

Place this after your established/related accept rule. See [E. Firewall](../E-firewall/).

## 4. Trunk to a second MikroTik (switch role)

On R2/SW1, with ether1 as the trunk from R1 and ether2 as an access port for VLAN 10:

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

## EVE-NG lab

```
[Cloud0] --ether1-- R1 --ether2 (VLAN10 access)-- PC1
                       --ether3 (VLAN20 access)-- PC2
                       --ether4 (trunk)-- SW1 (CHR) --ether2 (VLAN10 access)-- PC3
```

1. Build R1 as in section 3, with DHCP per VLAN.
2. Build SW1 as in section 4.
3. PC1 and PC3 (both VLAN 10) get `192.168.10.x` and can ping each other through the trunk.
4. PC2 gets `192.168.20.x`.
5. Ping PC1 → PC2: it works (routed by R1). Add the isolation rule, and the ping fails.
6. From R1, ping SW1's management IP `192.168.99.2`.

## Verify

```
/interface bridge vlan print            # current tagged/untagged per VLAN
/interface bridge port print            # PVIDs
/interface bridge host print            # MACs learned per VLAN
/interface vlan print
/ip address print
```

## Troubleshooting

| Problem | Fix |
|---|---|
| Lost access after `vlan-filtering=yes` | The management IP was on a port or VLAN not in the VLAN table, or `bridge1` wasn't tagged. Use Safe Mode, console or MAC WinBox. |
| VLAN interface gets no traffic | `bridge1` missing from `tagged=` for that VLAN. |
| Access port client gets nothing | Wrong PVID, or the port is missing from `untagged=`. |
| Trunk doesn't pass a VLAN | The VLAN ID isn't tagged on the trunk port on **both** sides. |
| `print` shows dynamic VLAN entries | Normal. RouterOS adds untagged entries from PVIDs automatically. |
| DHCP server not answering | It must be bound to the **VLAN interface**, not the bridge or port. |
