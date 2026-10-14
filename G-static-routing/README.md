# G. Static Routing

> **Status:** 📝 Not yet tested in lab

## Concept

The routing table decides where each packet goes. RouterOS picks a route by:
1. **Longest prefix match:** `/24` beats `/16`, which beats `/0`.
2. Then lowest **distance** (administrative distance).
3. Only **active** routes (gateway reachable) are used.

| Route source | Default distance |
|---|---|
| Connected | 0 |
| Static | 1 |
| eBGP | 20 |
| OSPF | 110 |
| iBGP | 200 |

### Route flags in `/ip route print`

| Flag | Meaning |
|---|---|
| `A` | Active, being used |
| `D` | Dynamic (connected, DHCP, OSPF, BGP) |
| `c` | Connected |
| `s` | Static |
| `o` | OSPF |
| `b` | BGP |
| `d` | DHCP |
| `+` | ECMP (multiple gateways) |

## 1. Basic static routes

```
/ip route add dst-address=0.0.0.0/0 gateway=192.168.1.1 comment="Default route"
/ip route add dst-address=10.2.0.0/24 gateway=10.0.12.2 comment="To R2 LAN"
/ip route print
/ip route print detail where dst-address=10.2.0.0/24
```

Gateway can be an IP or an interface (`gateway=ether2`, for point-to-point links like PPPoE or tunnels).

Blackhole (drop traffic to a range):

```
/ip route add dst-address=10.99.0.0/16 blackhole
```

## 2. Dual-WAN failover (distance + check-gateway)

```
/ip route add dst-address=0.0.0.0/0 gateway=192.168.1.1 distance=1 check-gateway=ping comment="ISP1 primary"
/ip route add dst-address=0.0.0.0/0 gateway=192.168.2.1 distance=2 comment="ISP2 backup"
```

`check-gateway=ping` pings the gateway every 10 seconds. After failures, the route turns inactive and ISP2 takes over.

**Limitation:** this only detects a dead **gateway**, not a dead internet behind a working modem.

## 3. Recursive failover (detects an upstream internet outage)

Check a real internet host through each ISP:

```
# Host routes: reach 8.8.8.8 only via ISP1, 1.1.1.1 only via ISP2
/ip route add dst-address=8.8.8.8/32 gateway=192.168.1.1 scope=10 comment="Monitor via ISP1"
/ip route add dst-address=1.1.1.1/32 gateway=192.168.2.1 scope=10 comment="Monitor via ISP2"

# Default routes resolve recursively through those hosts
/ip route add dst-address=0.0.0.0/0 gateway=8.8.8.8 target-scope=11 check-gateway=ping distance=1 comment="Default via ISP1"
/ip route add dst-address=0.0.0.0/0 gateway=1.1.1.1 target-scope=11 check-gateway=ping distance=2 comment="Default via ISP2"
```

If 8.8.8.8 stops answering through ISP1, the primary default route goes down and ISP2 takes over.

Side effect: your traffic to 8.8.8.8 always goes through ISP1. Pick monitor IPs you don't otherwise need.

## 4. Load balancing (ECMP)

```
/ip route add dst-address=0.0.0.0/0 gateway=192.168.1.1,192.168.2.1
```

RouterOS 7 balances per connection (hash-based). For more control, use PCC with mangle and routing tables (section 5).

## 5. Policy-based routing (routing tables)

Send VLAN 20 out ISP2, everyone else out ISP1:

```
/routing table add name=via-isp2 fib
/ip route add dst-address=0.0.0.0/0 gateway=192.168.2.1 routing-table=via-isp2
/routing rule add src-address=192.168.20.0/24 action=lookup-only-in-table table=via-isp2
```

Mangle-based (more flexible, e.g. by port):

```
/ip firewall mangle add chain=prerouting in-interface-list=LAN dst-address-type=!local protocol=tcp dst-port=443 action=mark-routing new-routing-mark=via-isp2 passthrough=no
```

Remember: masquerade must exist for **both** WAN interfaces.

## 6. Route to a LAN behind another router

```
R1 (LAN 10.1.0.0/24) --10.0.12.0/30-- R2 (LAN 10.2.0.0/24)
```

R1:
```
/ip address add address=10.0.12.1/30 interface=ether3
/ip route add dst-address=10.2.0.0/24 gateway=10.0.12.2
```

R2:
```
/ip address add address=10.0.12.2/30 interface=ether1
/ip route add dst-address=10.1.0.0/24 gateway=10.0.12.1
/ip route add dst-address=0.0.0.0/0 gateway=10.0.12.1      # R2 reaches the internet through R1
```

Routes must exist **in both directions**, or replies get lost.

## EVE-NG lab

```
[Cloud0] --ether1-- R1 --ether2-- PC1 (10.1.0.0/24)
                    |
                 ether3  10.0.12.0/30
                    |
                 ether1
                    R2 --ether2-- PC2 (10.2.0.0/24)
```

1. Address everything, with DHCP servers on both LANs.
2. Add the static routes from section 6 on R1 and R2.
3. PC1 ↔ PC2 ping works.
4. R1 needs masquerade for `10.2.0.0/24` too. Using `out-interface-list=WAN` covers it.
5. PC2 pings 8.8.8.8.
6. Remove one route and watch the ping fail. Run `trace` from VPCS to see where it stops.

Failover lab: add a second "ISP" by connecting R1 ether4 to a third CHR acting as ISP2. Then build section 2 and shut ISP1's link (`/interface disable ether1`).

## Verify

```
/ip route print
/ip route print where active
/ip route check 8.8.8.8          # which route would be used
/tool traceroute 10.2.0.10
/ping 10.2.0.10 src-address=10.1.0.1
```

## Troubleshooting

| Problem | Fix |
|---|---|
| Route shows but isn't active | Gateway unreachable: no connected route to it, or check-gateway failed. |
| One-way traffic | Missing return route on the other router. |
| Failover never switches | No `check-gateway`, or the modem stays up while the internet is down. Use recursive routing. |
| Recursive route inactive | Check `scope` / `target-scope` values: host route `scope=10`, default `target-scope=11`. |
| PBR not working | Fasttrack bypasses mangle. Also check the mangle counters and that the routing-table route exists. |
