# F. NAT

> **Status:** 📝 Not yet tested in lab

## Concept

**NAT (Network Address Translation)** rewrites the IP addresses in packets.

| Type | Chain | What it does | Typical use |
|---|---|---|---|
| **Masquerade** | `srcnat` | Replaces the source IP with the out-interface's IP (dynamic) | LAN → internet on DHCP/PPPoE WAN |
| **src-nat** | `srcnat` | Replaces the source IP with a fixed IP | LAN → internet on a static WAN IP |
| **dst-nat** | `dstnat` | Replaces the destination IP/port | Port forwarding to an internal server |
| **netmap** | either | 1:1 subnet mapping | Overlapping subnets, 1:1 NAT |
| **redirect** | `dstnat` | Sends to the router itself | Force DNS to the router |

NAT only processes the **first packet** of a connection. Connection tracking handles the rest automatically, including the replies.

## 1. Internet access for the LAN (masquerade)

```
/ip firewall nat add chain=srcnat out-interface-list=WAN action=masquerade comment="LAN to Internet"
```

You need the interface lists from [E. Firewall](../E-firewall/), or use `out-interface=ether1`.

If the WAN IP is **static**, `src-nat` is slightly more efficient:

```
/ip firewall nat add chain=srcnat out-interface=ether1 action=src-nat to-addresses=203.0.113.10
```

## 2. Port forwarding (dst-nat)

Forward WAN TCP 8080 to an internal web server at 192.168.10.50 on port 80:

```
/ip firewall nat add chain=dstnat in-interface-list=WAN protocol=tcp dst-port=8080 action=dst-nat to-addresses=192.168.10.50 to-ports=80 comment="Web server"
```

Forward RDP to a PC, restricted to one source:

```
/ip firewall nat add chain=dstnat in-interface-list=WAN protocol=tcp dst-port=3389 src-address-list=MGMT-ALLOWED action=dst-nat to-addresses=192.168.10.60
```

Port range:

```
/ip firewall nat add chain=dstnat in-interface-list=WAN protocol=udp dst-port=5060-5070 action=dst-nat to-addresses=192.168.20.10
```

The base forward-chain firewall rule `connection-nat-state=!dstnat` already **allows** dst-nat-ed traffic, so no extra filter rule is needed.

If the WAN IP is static, use `dst-address=<wan-ip>` instead of `in-interface-list=WAN`. That's required for hairpin NAT below.

## 3. Hairpin NAT (access the port forward from inside the LAN)

**Problem:** LAN PC → `public-ip:8080` fails, because the server replies directly to the PC, bypassing the router.

**Fix:** (1) make dst-nat match by destination IP rather than in-interface, and (2) masquerade LAN→LAN traffic toward the server:

```
/ip firewall nat add chain=dstnat dst-address=203.0.113.10 protocol=tcp dst-port=8080 action=dst-nat to-addresses=192.168.10.50 to-ports=80
/ip firewall nat add chain=srcnat src-address=192.168.10.0/24 dst-address=192.168.10.50 protocol=tcp dst-port=80 out-interface=bridge1 action=masquerade comment="Hairpin"
```

With a dynamic WAN IP, use `dst-address-type=local` plus `in-interface-list=LAN` on the dst-nat rule, or keep the public IP in an address list updated by script or DDNS (`/ip cloud`).

## 4. Force all DNS through the router

```
/ip firewall nat add chain=dstnat in-interface-list=LAN protocol=udp dst-port=53 action=redirect to-ports=53
/ip firewall nat add chain=dstnat in-interface-list=LAN protocol=tcp dst-port=53 action=redirect to-ports=53
```

## 5. 1:1 NAT with netmap

Map a public /29 to an internal /29:

```
/ip firewall nat add chain=dstnat dst-address=203.0.113.8/29 action=netmap to-addresses=192.168.50.8/29
/ip firewall nat add chain=srcnat src-address=192.168.50.8/29 action=netmap to-addresses=203.0.113.8/29
```

## 6. NAT bypass for VPN

When a site-to-site VPN uses IPsec policies, masquerade would rewrite the traffic before IPsec catches it. Add an accept rule **above** masquerade:

```
/ip firewall nat add chain=srcnat src-address=192.168.10.0/24 dst-address=192.168.2.0/24 action=accept place-before=0 comment="No NAT to VPN"
```

See [J. VPN](../J-vpn/).

## EVE-NG lab

```
[Cloud0 / home LAN 192.168.1.0/24] --ether1-- R1 --ether2-- PC1 (192.168.10.10)
                                                  \-ether3-- SRV (VPCS or CHR, 192.168.10.50)
```

Treat your home LAN as "the internet".

1. Add masquerade. PC1 can now `ping 8.8.8.8`.
2. Make a second CHR the "server": give it `192.168.10.50` and enable `/ip service set www disabled=no` so it serves WebFig on port 80.
3. Add a dst-nat from WAN TCP 8080 to `192.168.10.50:80`.
4. From your Windows PC browser, open `http://192.168.1.133:8080`. You should see the SRV WebFig page.
5. From PC1 that won't work by IP (VPCS has no browser), but test hairpin with a CHR client: `/tool fetch url="http://192.168.1.133:8080" mode=http`.

## Verify

```
/ip firewall nat print
/ip firewall nat print stats
/ip firewall connection print where dst-address~"192.168.10.50"
/tool torch interface=ether1 port=8080
```

## Troubleshooting

| Problem | Fix |
|---|---|
| LAN has no internet | Missing masquerade, wrong out-interface, or no default route on the router. |
| Port forward works from outside but not inside | You need hairpin NAT. |
| Port forward doesn't work at all | ISP CGNAT (the WAN IP is 100.64.x.x or private), the ISP blocks the port, the server's firewall blocks it, or the server has the wrong gateway. |
| NAT rule counters stay 0 | NAT only sees **new** connections. Check that the rule order and match fields are right. |
| VPN traffic gets NATed | Add the NAT bypass rule above masquerade. |
