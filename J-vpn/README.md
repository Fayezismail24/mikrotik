# J. VPN: WireGuard, IPsec, L2TP, SSTP

> **Status:** 📝 Not yet tested in lab

## Concept

A VPN builds an encrypted tunnel over an untrusted network.

| VPN | Use case | Pros | Cons |
|---|---|---|---|
| **WireGuard** | Site-to-site, road warrior | Fast, simple, modern crypto | No user/password auth (key-based only) |
| **IPsec IKEv2** | Site-to-site with any vendor | Industry standard | Complex config |
| **L2TP/IPsec** | Road warrior with built-in OS clients | Native client on Windows, macOS, iOS, Android | Older; NAT issues on some networks |
| **SSTP** | Road warrior through strict firewalls | Uses TCP 443, passes almost everything | TCP-over-TCP is slow; Windows-centric |
| **OpenVPN** | Road warrior with the OpenVPN client | Popular; v7 supports UDP | Slower than WireGuard |

**Recommendation:** use WireGuard unless you need interop with another vendor (IPsec) or native clients without an app (L2TP/IPsec).

## Lab topology

```
LAN-A 192.168.10.0/24 -- R1 ether1 (WAN 192.168.1.133) ---- [home LAN = "internet"] ---- ether1 (WAN 192.168.1.134) R2 -- LAN-B 192.168.20.0/24
```

Both WANs are on Cloud0, getting DHCP addresses from your home router. Note the real IPs they get.

---

## 1. WireGuard site-to-site

### R1

```
/interface wireguard add name=wg-s2s listen-port=13231
/interface wireguard print          # copy R1's public-key
/ip address add address=10.255.0.1/30 interface=wg-s2s
```

### R2

```
/interface wireguard add name=wg-s2s listen-port=13231
/interface wireguard print          # copy R2's public-key
/ip address add address=10.255.0.2/30 interface=wg-s2s
```

### Peers (each side gets the **other** side's public key)

R1:
```
/interface wireguard peers add interface=wg-s2s public-key="<R2-public-key>" endpoint-address=192.168.1.134 endpoint-port=13231 allowed-address=10.255.0.2/32,192.168.20.0/24 persistent-keepalive=25s
/ip route add dst-address=192.168.20.0/24 gateway=wg-s2s
```

R2:
```
/interface wireguard peers add interface=wg-s2s public-key="<R1-public-key>" endpoint-address=192.168.1.133 endpoint-port=13231 allowed-address=10.255.0.1/32,192.168.10.0/24 persistent-keepalive=25s
/ip route add dst-address=192.168.10.0/24 gateway=wg-s2s
```

### Firewall (both sides)

```
/ip firewall filter add chain=input protocol=udp dst-port=13231 action=accept comment="WireGuard" place-before=[find comment="IN: drop all not from LAN"]
```

Also add `wg-s2s` to the **LAN** interface list if you want tunnel traffic treated like LAN traffic.

`allowed-address` works as both an ACL and a routing hint: traffic from the peer is only accepted from those source IPs.

---

## 2. WireGuard road warrior (phone or laptop)

R1:
```
/interface wireguard add name=wg-rw listen-port=13232
/ip address add address=10.66.0.1/24 interface=wg-rw
/interface wireguard peers add interface=wg-rw public-key="<phone-public-key>" allowed-address=10.66.0.2/32 comment="Fayez phone"
/ip firewall filter add chain=input protocol=udp dst-port=13232 action=accept place-before=[find comment="IN: drop all not from LAN"]
/interface list member add list=LAN interface=wg-rw
```

Client config (WireGuard app):

```ini
[Interface]
PrivateKey = <phone-private-key>
Address = 10.66.0.2/32
DNS = 192.168.10.1

[Peer]
PublicKey = <R1-wg-rw-public-key>
Endpoint = <R1-public-IP-or-DDNS>:13232
AllowedIPs = 192.168.10.0/24, 10.66.0.0/24     # or 0.0.0.0/0 for full tunnel
PersistentKeepalive = 25
```

RouterOS 7.11+ can generate client configs: `/interface wireguard peers show-client-config [find comment="Fayez phone"]`. Also check `/ip cloud set ddns-enabled=yes` for a free DDNS name.

---

## 3. IPsec IKEv2 site-to-site (PSK)

### R1

```
/ip ipsec profile add name=s2s-prof hash-algorithm=sha256 enc-algorithm=aes-256 dh-group=modp2048
/ip ipsec proposal add name=s2s-prop auth-algorithms=sha256 enc-algorithms=aes-256-cbc pfs-group=modp2048
/ip ipsec peer add name=to-R2 address=192.168.1.134/32 exchange-mode=ike2 profile=s2s-prof
/ip ipsec identity add peer=to-R2 auth-method=pre-shared-key secret="IpsecPSK!2026"
/ip ipsec policy add src-address=192.168.10.0/24 dst-address=192.168.20.0/24 tunnel=yes peer=to-R2 proposal=s2s-prop action=encrypt
/ip firewall nat add chain=srcnat src-address=192.168.10.0/24 dst-address=192.168.20.0/24 action=accept place-before=0 comment="No NAT to R2 LAN"
```

### R2: the mirror image

```
/ip ipsec profile add name=s2s-prof hash-algorithm=sha256 enc-algorithm=aes-256 dh-group=modp2048
/ip ipsec proposal add name=s2s-prop auth-algorithms=sha256 enc-algorithms=aes-256-cbc pfs-group=modp2048
/ip ipsec peer add name=to-R1 address=192.168.1.133/32 exchange-mode=ike2 profile=s2s-prof
/ip ipsec identity add peer=to-R1 auth-method=pre-shared-key secret="IpsecPSK!2026"
/ip ipsec policy add src-address=192.168.20.0/24 dst-address=192.168.10.0/24 tunnel=yes peer=to-R1 proposal=s2s-prop action=encrypt
/ip firewall nat add chain=srcnat src-address=192.168.20.0/24 dst-address=192.168.10.0/24 action=accept place-before=0 comment="No NAT to R1 LAN"
```

### Firewall (both sides): allow IKE and ESP

```
/ip firewall filter add chain=input protocol=udp dst-port=500,4500 action=accept place-before=[find comment="IN: drop all not from LAN"]
/ip firewall filter add chain=input protocol=ipsec-esp action=accept place-before=[find comment="IN: drop all not from LAN"]
```

### Fasttrack exception

Fasttracked packets can bypass IPsec policies. Exclude IPsec traffic:

```
/ip firewall filter set [find action=fasttrack-connection] ipsec-policy=out,none
```

Or add accept rules with `ipsec-policy=in,ipsec` and `out,ipsec` above fasttrack.

### Verify IPsec

```
/ip ipsec active-peers print
/ip ipsec installed-sa print
/ip ipsec policy print stats
```

---

## 4. L2TP/IPsec road warrior (native OS clients)

```
/ip pool add name=vpn-pool ranges=10.77.0.10-10.77.0.50
/ppp profile add name=l2tp-prof local-address=10.77.0.1 remote-address=vpn-pool dns-server=192.168.10.1 use-encryption=required
/interface l2tp-server server set enabled=yes default-profile=l2tp-prof use-ipsec=required ipsec-secret="L2tpIpsecKey!" authentication=mschap2
/ppp secret add name=fayez password="UserPass!" service=l2tp profile=l2tp-prof
/ip firewall filter add chain=input protocol=udp dst-port=500,1701,4500 action=accept place-before=[find comment="IN: drop all not from LAN"]
/ip firewall filter add chain=input protocol=ipsec-esp action=accept place-before=[find comment="IN: drop all not from LAN"]
```

Windows client: Settings → VPN → Add. Type **L2TP/IPsec with pre-shared key**, enter the key and the username/password.

To let VPN clients reach the LAN and get treated as LAN, add `/interface list member add list=LAN interface=all-ppp`, or create a rule for 10.77.0.0/24.

---

## 5. SSTP road warrior (TCP 443)

Create certificates on the router:

```
/certificate add name=ca common-name=MyCA key-usage=key-cert-sign,crl-sign
/certificate sign ca
/certificate add name=sstp-server common-name=<public-IP-or-DNS>
/certificate sign sstp-server ca=ca
/certificate export-certificate ca          # import ca.crt to the client's Trusted Root store
```

Server:

```
/interface sstp-server server set enabled=yes certificate=sstp-server default-profile=l2tp-prof authentication=mschap2
/ppp secret add name=fayez-sstp password="UserPass!" service=sstp profile=l2tp-prof
/ip firewall filter add chain=input protocol=tcp dst-port=443 action=accept place-before=[find comment="IN: drop all not from LAN"]
```

If WebFig also uses 443 (`www-ssl`), move one of them to a different port.

---

## EVE-NG lab

1. Build R1 and R2 with LANs plus a VPCS on each. Put both WANs on Cloud0.
2. Base firewall and NAT on both.
3. **WireGuard s2s:** PC-A pings PC-B.
4. Remove WireGuard and build **IPsec IKEv2** for the same goal. Check `installed-sa`.
5. **Road warrior:** install the WireGuard app on your Windows PC and connect to R1 at `192.168.1.133:13232`. Then ping PC-A (192.168.10.x).

## Troubleshooting

| Problem | Fix |
|---|---|
| WireGuard handshake never happens | Wrong public key (each side needs the **other's**), firewall blocks UDP, wrong endpoint. Check `/interface wireguard peers print detail` for `last-handshake`. |
| Handshake OK but no traffic | `allowed-address` is missing the remote subnet, or there's no route to the remote LAN via the WG interface. |
| IPsec phase 1 fails | PSK, profile (hash/enc/DH) or peer address mismatch. `/log print where topics~"ipsec"` |
| IPsec SA up but no traffic | Missing NAT bypass, or fasttrack bypassing the policy. |
| L2TP connects, no LAN access | Client traffic is dropped by the forward chain, or the client lacks a route. Add `all-ppp` to the LAN list. |
| SSTP certificate error | The client doesn't trust the CA, or the CN doesn't match the address used. |
