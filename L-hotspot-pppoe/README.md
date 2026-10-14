# L. Hotspot and PPPoE

> **Status:** 📝 Not yet tested in lab

## Concept

| | Hotspot | PPPoE |
|---|---|---|
| What | Captive portal: a login page in the browser | Point-to-point tunnel over Ethernet with username/password |
| Where used | Cafés, hotels, guest Wi-Fi | ISPs and WISPs: customer CPE dials into a concentrator |
| Client needs | Just a browser | A PPPoE client (router, or OS dialer) |
| Per-user limits | User profiles (rate-limit, time, data) | PPP profiles (rate-limit, IP pool) |
| Central auth | RADIUS (User Manager or external) | RADIUS |

---

## Part 1: Hotspot

### Setup wizard (recommended first time)

```
/ip hotspot setup
```

It asks for:
1. Interface: `ether2` (or the bridge/VLAN for guests)
2. Local address: `192.168.50.1/24`
3. Masquerade network: `yes`
4. Address pool: `192.168.50.10-192.168.50.254`
5. SSL certificate: `none` (for the lab)
6. SMTP server: `0.0.0.0`
7. DNS servers: `1.1.1.1`
8. DNS name: `hotspot.lab` (the portal URL)
9. Local user: `admin` / password

The wizard creates: the IP, pool, DHCP server, hotspot server, profile, NAT and firewall rules, and walled-garden basics.

### Users and profiles

```
/ip hotspot user profile add name=1hour rate-limit=2M/5M session-timeout=1h shared-users=1
/ip hotspot user profile add name=vip rate-limit=10M/20M shared-users=3
/ip hotspot user add name=guest1 password=1234 profile=1hour
/ip hotspot user add name=boss password=Secret! profile=vip limit-bytes-total=5G
```

`rate-limit` format: `rx/tx` from the router's view, which works out to **upload/download** for the client.

### Walled garden (allowed without login)

```
/ip hotspot walled-garden add dst-host=*.mycompany.com
/ip hotspot walled-garden ip add dst-address=192.168.50.5 action=accept
```

### Bypass a device (printer, IP camera)

```
/ip hotspot ip-binding add mac-address=AA:BB:CC:DD:EE:FF type=bypassed comment="Printer"
```

### Customize the login page

Download `/hotspot/login.html` from **Files** in WinBox, edit it, and upload it back.

### Monitor

```
/ip hotspot active print
/ip hotspot host print
/ip hotspot user print stats
/ip hotspot active remove [find user=guest1]    # kick a user
```

### User Manager (RADIUS, vouchers)

RouterOS 7 includes **User Manager** as an extra package (`user-manager`). It provides vouchers, prepaid plans and many routers authenticating against one database. Link the hotspot to it:

```
/radius add service=hotspot address=127.0.0.1 secret=RadSecret
/ip hotspot profile set [find] use-radius=yes
/user-manager set enabled=yes
/user-manager router add name=local address=127.0.0.1 shared-secret=RadSecret
```

---

## Part 2: PPPoE server (ISP side)

```
/ip pool add name=pppoe-pool ranges=10.100.0.10-10.100.0.254
/ppp profile add name=plan-10M local-address=10.100.0.1 remote-address=pppoe-pool dns-server=1.1.1.1,8.8.8.8 rate-limit=5M/10M only-one=yes
/ppp profile add name=plan-50M local-address=10.100.0.1 remote-address=pppoe-pool dns-server=1.1.1.1,8.8.8.8 rate-limit=20M/50M only-one=yes
/interface pppoe-server server add service-name=ISP interface=ether2 default-profile=plan-10M authentication=pap,chap,mschap2 one-session-per-host=yes disabled=no
/ppp secret add name=customer1 password=Cust1Pass service=pppoe profile=plan-10M
/ppp secret add name=customer2 password=Cust2Pass service=pppoe profile=plan-50M remote-address=10.100.0.50   # static IP
```

The PPPoE server interface (`ether2`) **should not have an IP** for customers. They get IPs over PPP.

Masquerade for customers:

```
/ip firewall nat add chain=srcnat src-address=10.100.0.0/24 out-interface-list=WAN action=masquerade
```

`rate-limit` in a PPP profile automatically creates a dynamic simple queue per user. Disable fasttrack for the limits to apply precisely.

### Monitor

```
/ppp active print
/interface pppoe-server print
/ppp active remove [find name=customer1]    # disconnect
```

### Set the MTU/MSS (avoids broken websites)

```
/ppp profile set [find] change-tcp-mss=yes
/interface pppoe-server server set [find] max-mtu=1480 max-mru=1480
```

---

## Part 3: PPPoE client (customer side, or your home router with an ISP)

```
/interface pppoe-client add name=pppoe-out1 interface=ether1 user=customer1 password=Cust1Pass add-default-route=yes use-peer-dns=yes disabled=no
/interface pppoe-client monitor pppoe-out1
/interface list member add list=WAN interface=pppoe-out1
```

Masquerade must use `out-interface-list=WAN`, which now includes `pppoe-out1`.

If the ISP uses a VLAN for PPPoE (common with fiber):

```
/interface vlan add interface=ether1 name=vlan35-wan vlan-id=35
/interface pppoe-client set pppoe-out1 interface=vlan35-wan
```

---

## EVE-NG lab

### Hotspot lab

```
[Cloud0] --ether1-- R1 --ether2-- Windows/Linux VM or a CHR (needs a browser to see the portal)
```

VPCS has no browser. Use a lightweight Linux or Windows node in EVE-NG, or test with a CHR: `/tool fetch url=http://example.com` should redirect.

1. Run `/ip hotspot setup` on ether2.
2. Add `guest1` with profile `1hour`.
3. On the client, browse anywhere and you get redirected to the login page. Log in and browsing works.
4. Check `/ip hotspot active print`.

### PPPoE lab

```
[Cloud0] --ether1-- ISP-R1 --ether2-- CUST-R2 --ether2-- PC1
```

1. On ISP-R1: pool, profile, PPPoE server on ether2, secrets, masquerade.
2. On CUST-R2: PPPoE client on ether1, then LAN + DHCP + masquerade on ether2.
3. PC1 pings 8.8.8.8. The path is PC1 → CUST-R2 (NAT) → PPPoE → ISP-R1 (NAT) → internet.
4. On ISP-R1, `/ppp active print` shows customer1, and `/queue simple print` shows the dynamic queue.

## Troubleshooting

| Problem | Fix |
|---|---|
| Hotspot login page never shows | The client uses HTTPS-only sites (try `http://neverssl.com`), or DNS is not via the router. |
| Hotspot users can't browse after login | Missing NAT, or the hotspot interface isn't the one clients are on (a bridge vs. a port). |
| PPPoE client stuck "connecting" | Wrong service-name or interface, the server is on another interface, or VLAN tagging is needed. |
| PPPoE auth fails | Username/password, or `authentication=` methods don't match. `/log print where topics~"ppp"` |
| Some sites don't load over PPPoE | MTU/MSS. Set `change-tcp-mss=yes`. |
