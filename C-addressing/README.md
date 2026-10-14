# C. Addressing: IP, DHCP, DNS

> **Status:** 📝 Not yet tested in lab

## Concept

- **IP address:** RouterOS assigns addresses to interfaces with a prefix, e.g. `192.168.10.1/24`. Adding an address automatically creates a **connected route** for that subnet.
- **DHCP client:** gets an IP, gateway and DNS from an upstream router or ISP.
- **DHCP server:** hands out IPs to LAN clients. It needs three parts: an **IP pool**, a **network** (gateway/DNS to give out), and a **server** bound to an interface.
- **DNS:** the router resolves names for itself and, with `allow-remote-requests=yes`, acts as a DNS cache for the LAN.

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

## 1. Static IP addresses

```
/ip address add address=192.168.10.1/24 interface=ether2 comment="LAN"
/ip address print
/ip address remove [find address="192.168.10.1/24"]
```

Loopback-style address (useful for router IDs and management):

```
/interface bridge add name=loopback0
/ip address add address=10.255.255.1/32 interface=loopback0
```

RouterOS 7 also has a built-in `lo` interface you can use.

## 2. DHCP client (WAN side)

```
/ip dhcp-client add interface=ether1 add-default-route=yes use-peer-dns=yes use-peer-ntp=yes disabled=no
/ip dhcp-client print detail
/ip dhcp-client renew [find interface=ether1]
/ip dhcp-client release [find interface=ether1]
```

`default-route-distance=` controls route priority when you have two WANs. See [G. Static routing](../G-static-routing/).

## 3. DHCP server (LAN side)

### Option A: wizard

```
/ip dhcp-server setup
```

It asks for the interface, network, gateway, pool, DNS and lease time.

### Option B: manual (know what the wizard builds)

```
/ip pool add name=pool-lan ranges=192.168.10.100-192.168.10.200
/ip dhcp-server network add address=192.168.10.0/24 gateway=192.168.10.1 dns-server=192.168.10.1 domain=lab.local
/ip dhcp-server add name=dhcp-lan interface=ether2 address-pool=pool-lan lease-time=1d disabled=no
```

### Leases

```
/ip dhcp-server lease print
/ip dhcp-server lease make-static [find address=192.168.10.150]   # reservation from an existing lease
/ip dhcp-server lease add address=192.168.10.20 mac-address=AA:BB:CC:DD:EE:FF server=dhcp-lan comment="Printer"
```

### DHCP options (example: option 66 for VoIP phones)

```
/ip dhcp-server option add name=tftp code=66 value="'192.168.10.5'"
/ip dhcp-server network set [find address=192.168.10.0/24] dhcp-option=tftp
```

String values need the inner single quotes.

## 4. DNS

```
/ip dns set servers=1.1.1.1,8.8.8.8 allow-remote-requests=yes cache-size=4096KiB
/ip dns print
/ip dns cache print
/ip dns cache flush
```

**Security:** `allow-remote-requests=yes` makes the router answer DNS on **every** interface, including WAN. Always block UDP/TCP 53 from WAN in the firewall (see [E. Firewall](../E-firewall/)), or attackers will use you for DNS amplification.

### Static DNS entries

```
/ip dns static add name=router.lab.local address=192.168.10.1
/ip dns static add name=nas.lab.local address=192.168.10.20
/ip dns static add regexp=".*\\.ads\\.example\\.com" type=NXDOMAIN   # simple blocking
```

### DNS over HTTPS (DoH)

```
/ip dns set use-doh-server=https://cloudflare-dns.com/dns-query verify-doh-cert=yes
```

`verify-doh-cert=yes` needs a CA bundle imported into `/certificate`. For a lab you can set `verify-doh-cert=no` to test.

## 5. ARP

```
/ip arp print
/ip arp add address=192.168.10.20 mac-address=AA:BB:CC:DD:EE:FF interface=ether2   # static ARP
```

Setting `arp=reply-only` on an interface, together with DHCP's `add-arp=yes`, gives you "only DHCP clients can talk":

```
/ip dhcp-server set dhcp-lan add-arp=yes
/interface ethernet set ether2 arp=reply-only
```

## EVE-NG lab

**Topology:**

```
[Cloud0/home LAN] --ether1-- R1 --ether2-- [VPCS PC1]
```

1. R1 ether1: DHCP client (it already has one).
2. R1 ether2: `192.168.10.1/24`.
3. Build the DHCP server manually (Option B).
4. DNS: `servers=1.1.1.1`, `allow-remote-requests=yes`.
5. On PC1 (VPCS): `ip dhcp`, then `show ip`. It should get `192.168.10.100` or similar.
6. Make PC1's lease static.
7. Add a static DNS entry `r1.lab.local`. PC1 can't resolve names, but `/ping r1.lab.local` from R1 works.

PC1 can reach R1 now, but **not the internet** yet. That needs NAT, covered in [F. NAT](../F-nat/).

## Verify

```
/ip address print
/ip route print                    # connected routes (DAc)
/ip dhcp-server print
/ip dhcp-server lease print
/ip dns print
/ping 192.168.10.100
/resolve google.com
```

## Troubleshooting

| Problem | Fix |
|---|---|
| Client gets no lease | Server on the wrong interface (on a bridge? It must bind to the **bridge**, not the port), server disabled, or pool exhausted. |
| Client gets an IP but no internet | Missing NAT masquerade or default route. |
| Clients can't resolve names | `allow-remote-requests=no`, or `dns-server=` missing in `/ip dhcp-server network`. |
| Two DHCP servers on the LAN | A rogue device. Use `/ip dhcp-server alert` to detect it. |
| `/resolve` fails on the router | No DNS servers, or the WAN has no route. |
