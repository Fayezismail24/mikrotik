# H. Dynamic Routing: OSPF and BGP

> **Status:** 📝 Not yet tested in lab

## Concept

Static routes don't scale and don't react to failures on their own. Dynamic routing protocols let routers **share routes automatically**.

| | OSPF | BGP |
|---|---|---|
| Type | Link-state IGP | Path-vector EGP |
| Used for | Inside one organization | Between organizations and ISPs, or large internal networks |
| Picks paths by | Lowest cost (bandwidth-based) | Policy (AS path, local-pref, MED...) |
| Convergence | Fast (seconds) | Slower, but very controllable |
| Distance in RouterOS | 110 | 20 (eBGP), 200 (iBGP) |

> RouterOS 7 rewrote OSPF and BGP. The v6 commands (`/routing ospf network`, `/routing bgp peer`) **don't exist** in v7. Everything here is v7 syntax.

## Lab topology (used for both)

```
             10.0.12.0/30            10.0.23.0/30
  R1 ether2 ------------ ether1 R2 ether2 ------------ ether1 R3
  |                             |                             |
  lo 1.1.1.1/32                 lo 2.2.2.2/32                 lo 3.3.3.3/32
  ether3: 10.1.0.0/24 LAN       ether3: 10.2.0.0/24 LAN       ether3: 10.3.0.0/24 LAN
  ether1: Cloud0 (internet)

  Optional: R1 ether4 -- 10.0.13.0/30 -- R3 ether4 (redundant path)
```

Base addressing on R1 (repeat the pattern on R2 and R3):

```
/ip address add address=1.1.1.1/32 interface=lo
/ip address add address=10.0.12.1/30 interface=ether2
/ip address add address=10.1.0.1/24 interface=ether3
```

---

## Part 1: OSPF

### Key ideas

- **Router ID:** a unique 32-bit ID. Use the loopback IP.
- **Area 0 (backbone):** every other area must connect to it.
- **Neighbors** form on interfaces in the same area, subnet, hello/dead timers and MTU.
- **Passive interface:** advertises the subnet but sends no hellos. Use it on LAN ports.
- **Network type:** use `ptp` on /30 point-to-point links. That's faster and skips the DR/BDR election.

### Config: R1

```
/routing ospf instance add name=ospf1 version=2 router-id=1.1.1.1 redistribute=connected originate-default=if-installed
/routing ospf area add name=backbone area-id=0.0.0.0 instance=ospf1
/routing ospf interface-template add area=backbone interfaces=ether2 type=ptp
/routing ospf interface-template add area=backbone interfaces=ether3,lo passive
```

- `originate-default=if-installed`: R1 has the internet, so it advertises a default route when it has one.
- `redistribute=connected` is optional when every interface is already in a template. Shown here as an example.

### Config: R2

```
/routing ospf instance add name=ospf1 version=2 router-id=2.2.2.2
/routing ospf area add name=backbone area-id=0.0.0.0 instance=ospf1
/routing ospf interface-template add area=backbone interfaces=ether1,ether2 type=ptp
/routing ospf interface-template add area=backbone interfaces=ether3,lo passive
```

### Config: R3

```
/routing ospf instance add name=ospf1 version=2 router-id=3.3.3.3
/routing ospf area add name=backbone area-id=0.0.0.0 instance=ospf1
/routing ospf interface-template add area=backbone interfaces=ether1 type=ptp
/routing ospf interface-template add area=backbone interfaces=ether3,lo passive
```

You can also match templates by subnet instead of interface: `networks=10.0.0.0/8`.

### Cost (path preference)

```
/routing ospf interface-template set [find interfaces=ether4] cost=100
```

Lower cost wins. Use it on the redundant R1–R3 link to make it a backup.

### Authentication (MD5)

```
/routing ospf interface-template set [find interfaces=ether2] auth=md5 auth-id=1 auth-key="OspfKey123"
```

Both sides must match.

### Verify OSPF

```
/routing ospf neighbor print          # state must be Full
/routing ospf interface print
/routing ospf lsa print
/ip route print where ospf
```

### OSPF troubleshooting

| Problem | Fix |
|---|---|
| No neighbor | Different area, subnet, network type or MTU. Firewall blocking protocol 89. Interface passive by mistake. |
| Stuck in ExStart/Exchange | MTU mismatch. |
| Neighbor up but no routes | LAN interfaces not in any template, or no redistribute. |
| Duplicate router-id | Every router needs a unique `router-id`. |

---

## Part 2: BGP

### Key ideas

- **AS (Autonomous System):** a number identifying a network. Use private ASNs 64512–65534 for labs.
- **eBGP:** between different AS. **iBGP:** within the same AS.
- BGP doesn't discover neighbors. You configure each **connection** manually (TCP 179).
- BGP only advertises what you tell it to: networks, or redistributed routes, filtered by policy.

### Scenario: eBGP between three ASes

| Router | AS |
|---|---|
| R1 | 65001 |
| R2 | 65002 |
| R3 | 65003 |

Each router advertises its own LAN.

### Config: R1

```
/ip firewall address-list add list=bgp-out address=10.1.0.0/24
/routing bgp connection add name=to-R2 as=65001 router-id=1.1.1.1 local.role=ebgp remote.address=10.0.12.2 remote.as=65002 output.network=bgp-out
```

### Config: R2

```
/ip firewall address-list add list=bgp-out address=10.2.0.0/24
/routing bgp connection add name=to-R1 as=65002 router-id=2.2.2.2 local.role=ebgp remote.address=10.0.12.1 remote.as=65001 output.network=bgp-out
/routing bgp connection add name=to-R3 as=65002 router-id=2.2.2.2 local.role=ebgp remote.address=10.0.23.2 remote.as=65003 output.network=bgp-out
```

### Config: R3

```
/ip firewall address-list add list=bgp-out address=10.3.0.0/24
/routing bgp connection add name=to-R2 as=65003 router-id=3.3.3.3 local.role=ebgp remote.address=10.0.23.1 remote.as=65002 output.network=bgp-out
```

`output.network=` advertises the subnets in that address list, but **only if a matching route exists** in the routing table. Connected routes count.

### Templates (shared settings)

```
/routing bgp template add name=ebgp-default as=65002 router-id=2.2.2.2
/routing bgp connection add name=to-R1 templates=ebgp-default local.role=ebgp remote.address=10.0.12.1 remote.as=65001
```

### Route filters (v7 routing filters)

Accept only /24s from the 10.0.0.0/8 range from R1:

```
/routing filter rule add chain=from-R1 rule="if (dst in 10.0.0.0/8 && dst-len == 24) { accept } else { reject }"
/routing bgp connection set to-R1 input.filter=from-R1
```

Prefer a path with local-pref (for iBGP setups):

```
/routing filter rule add chain=from-ISP1 rule="set bgp-local-pref 200; accept"
```

Prepend the AS path (make your path less attractive to others):

```
/routing filter rule add chain=to-ISP2 rule="set bgp-path-prepend 3; accept"
```

### MD5 password

```
/routing bgp connection set to-R2 tcp-md5-key="BgpKey123"
```

### Verify BGP

```
/routing bgp connection print
/routing bgp session print            # state = established
/routing bgp advertisements print
/ip route print where bgp
/routing route print where bgp        # full detail including attributes
```

### BGP troubleshooting

| Problem | Fix |
|---|---|
| Session not established | Wrong `remote.address` or `remote.as`, no IP reachability, firewall blocking TCP 179 (allow it in the **input** chain). |
| Established but nothing received | The peer's `output.network` list is empty, or there's no matching route on the peer. |
| Routes received but not active | No route to the BGP next-hop. |
| Filter changes not applied | Resend: `/routing bgp session refresh [find]`. |

---

## EVE-NG lab steps

1. Build three CHR nodes R1–R3 with VPCS on each LAN. Link them as in the topology diagram.
2. Do the base addressing only (no static routes).
3. **OSPF lab:** configure OSPF and check the neighbors are Full. All three PCs should ping each other, and each router should learn a default from R1.
4. Add the R1–R3 redundant link with `cost=100`. Disable R1–R2 (`/interface disable ether2`) and watch the traffic reroute.
5. **BGP lab:** remove OSPF (`/routing ospf instance remove [find]`) and configure eBGP. All PCs ping each other again.
6. Add a routing filter on R2 that rejects `10.3.0.0/24` toward R1. R1 loses that route.

## Firewall note

If you run the base firewall from [E. Firewall](../E-firewall/), routing protocol traffic between routers hits the **input** chain. Allow it before the drop rule:

```
/ip firewall filter add chain=input protocol=ospf action=accept place-before=[find comment="IN: drop all not from LAN"]
/ip firewall filter add chain=input protocol=tcp dst-port=179 action=accept place-before=[find comment="IN: drop all not from LAN"]
```
