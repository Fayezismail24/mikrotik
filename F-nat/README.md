# F. NAT

> **Status:** 📝 Not yet tested in lab

## Concept

**NAT** rewrites IP addresses in packets.

| Type | Chain | What it does | Typical use |
|---|---|---|---|
| **Masquerade** | `srcnat` | Source → the out-interface's IP (dynamic) | LAN → internet on DHCP/PPPoE WAN |
| **src-nat** | `srcnat` | Source → a fixed IP | LAN → internet on a static WAN |
| **dst-nat** | `dstnat` | Destination → an internal IP/port | Port forwarding |
| **netmap** | either | 1:1 subnet mapping | 1:1 NAT, overlapping subnets |
| **redirect** | `dstnat` | Sends to the router itself | Force DNS to the router |

NAT only processes the **first packet** of a connection. Connection tracking handles the rest.

🖱️ **In WinBox,** NAT rules live at **IP → Firewall → NAT**. They use the same rule dialog as filter rules: **General** tab for matching, **Action** tab for the action plus **To Addresses** / **To Ports**.

---

## 1. Internet access for the LAN (masquerade)

🖱️ **WinBox:** IP → Firewall → **NAT** → **+**:
- General tab: Chain `srcnat`, Out. Interface List `WAN`.
- Action tab: `masquerade`.
- Comment: `LAN to Internet`.

Click **OK**.

⌨️ **CLI:**
```
/ip firewall nat add chain=srcnat out-interface-list=WAN action=masquerade comment="LAN to Internet"
```

### Static WAN IP: src-nat

🖱️ **WinBox:** NAT → **+**:
- General: Chain `srcnat`, Out. Interface `ether1`.
- Action: `src-nat`, To Addresses `203.0.113.10`.

⌨️ **CLI:**
```
/ip firewall nat add chain=srcnat out-interface=ether1 action=src-nat to-addresses=203.0.113.10
```

---

## 2. Port forwarding (dst-nat)

WAN TCP 8080 → internal web server `192.168.10.50:80`.

🖱️ **WinBox:** NAT → **+**:
- General: Chain `dstnat`, Protocol `tcp`, Dst. Port `8080`, In. Interface List `WAN`.
- Action: `dst-nat`, To Addresses `192.168.10.50`, To Ports `80`.
- Comment: `Web server`.

⌨️ **CLI:**
```
/ip firewall nat add chain=dstnat in-interface-list=WAN protocol=tcp dst-port=8080 action=dst-nat to-addresses=192.168.10.50 to-ports=80 comment="Web server"
```

### Restricted to a trusted source (RDP example)

🖱️ **WinBox:** NAT → **+**:
- General: Chain `dstnat`, Protocol `tcp`, Dst. Port `3389`, In. Interface List `WAN`.
- Advanced: Src. Address List `MGMT-ALLOWED`.
- Action: `dst-nat`, To Addresses `192.168.10.60`.

⌨️ **CLI:**
```
/ip firewall nat add chain=dstnat in-interface-list=WAN protocol=tcp dst-port=3389 src-address-list=MGMT-ALLOWED action=dst-nat to-addresses=192.168.10.60
```

### Port range

🖱️ **WinBox:** same as above, with Protocol `udp`, Dst. Port `5060-5070`, To Addresses `192.168.20.10`.

⌨️ **CLI:**
```
/ip firewall nat add chain=dstnat in-interface-list=WAN protocol=udp dst-port=5060-5070 action=dst-nat to-addresses=192.168.20.10
```

The base forward-chain rule `connection-nat-state=!dstnat` already **allows** forwarded traffic, so no extra filter rule is needed.

---

## 3. Hairpin NAT (reach the port forward from inside the LAN)

**Problem:** a LAN PC going to `public-ip:8080` fails, because the server replies directly to the PC and bypasses the router.

### Step 1: dst-nat matched by destination IP

🖱️ **WinBox:** NAT → **+**:
- General: Chain `dstnat`, Dst. Address `203.0.113.10`, Protocol `tcp`, Dst. Port `8080`.
- Action: `dst-nat`, To Addresses `192.168.10.50`, To Ports `80`.

⌨️ **CLI:**
```
/ip firewall nat add chain=dstnat dst-address=203.0.113.10 protocol=tcp dst-port=8080 action=dst-nat to-addresses=192.168.10.50 to-ports=80
```

### Step 2: masquerade LAN → server

🖱️ **WinBox:** NAT → **+**:
- General: Chain `srcnat`, Src. Address `192.168.10.0/24`, Dst. Address `192.168.10.50`, Protocol `tcp`, Dst. Port `80`, Out. Interface `bridge1`.
- Action: `masquerade`.
- Comment: `Hairpin`.

⌨️ **CLI:**
```
/ip firewall nat add chain=srcnat src-address=192.168.10.0/24 dst-address=192.168.10.50 protocol=tcp dst-port=80 out-interface=bridge1 action=masquerade comment="Hairpin"
```

With a dynamic WAN IP, keep the public IP in an address list (a script, see [N](../N-scripting/)) and match on Dst. Address List instead.

---

## 4. Force all DNS through the router

🖱️ **WinBox:** NAT → **+** (twice, once for `udp` and once for `tcp`):
- General: Chain `dstnat`, Protocol `udp` (then `tcp`), Dst. Port `53`, In. Interface List `LAN`.
- Action: `redirect`, To Ports `53`.

⌨️ **CLI:**
```
/ip firewall nat add chain=dstnat in-interface-list=LAN protocol=udp dst-port=53 action=redirect to-ports=53
/ip firewall nat add chain=dstnat in-interface-list=LAN protocol=tcp dst-port=53 action=redirect to-ports=53
```

---

## 5. 1:1 NAT with netmap

🖱️ **WinBox:** NAT → **+**:
- Rule 1: Chain `dstnat`, Dst. Address `203.0.113.8/29`. Action `netmap`, To Addresses `192.168.50.8/29`.
- Rule 2: Chain `srcnat`, Src. Address `192.168.50.8/29`. Action `netmap`, To Addresses `203.0.113.8/29`.

⌨️ **CLI:**
```
/ip firewall nat add chain=dstnat dst-address=203.0.113.8/29 action=netmap to-addresses=192.168.50.8/29
/ip firewall nat add chain=srcnat src-address=192.168.50.8/29 action=netmap to-addresses=203.0.113.8/29
```

---

## 6. NAT bypass for VPN

Put an accept rule **above** masquerade so IPsec traffic isn't NATed.

🖱️ **WinBox:** NAT → **+**:
- General: Chain `srcnat`, Src. Address `192.168.10.0/24`, Dst. Address `192.168.2.0/24`.
- Action: `accept`.
- Comment: `No NAT to VPN`.

Click OK, then **drag the rule to the top**.

⌨️ **CLI:**
```
/ip firewall nat add chain=srcnat src-address=192.168.10.0/24 dst-address=192.168.2.0/24 action=accept place-before=0 comment="No NAT to VPN"
```

---

## EVE-NG lab

```
[Cloud0 / home LAN 192.168.1.0/24] --ether1-- R1 --ether2-- PC1 (192.168.10.10)
                                                  \-ether3-- SRV (CHR, 192.168.10.50)
```

1. Add masquerade. PC1 can now `ping 8.8.8.8`.
2. On SRV: IP → Services → enable `www` (port 80).
3. On R1: dst-nat from WAN TCP 8080 to `192.168.10.50:80`.
4. From your Windows browser, open `http://192.168.1.133:8080`. You should see SRV's WebFig.
5. Test hairpin from a CHR on the LAN: `/tool fetch url="http://192.168.1.133:8080" mode=http`.

## Verify

| Check | 🖱️ WinBox | ⌨️ CLI |
|---|---|---|
| NAT rules + counters | IP → Firewall → NAT | `/ip firewall nat print stats` |
| Translated connections | IP → Firewall → Connections (filter by dst) | `/ip firewall connection print where dst-address~"192.168.10.50"` |
| Live traffic on the port | Tools → Torch → Interface `ether1`, Port `8080` | `/tool torch interface=ether1 port=8080` |

## Troubleshooting

| Problem | Fix |
|---|---|
| LAN has no internet | Missing masquerade, wrong out-interface, or no default route. |
| Forward works outside, not inside | You need hairpin NAT. |
| Forward doesn't work at all | ISP CGNAT (WAN is 100.64.x.x or private), ISP blocks the port, the server's firewall, or the server has the wrong gateway. |
| Counters stay 0 | NAT only sees **new** connections. Check the match fields. |
| VPN traffic NATed | Put the NAT bypass rule above masquerade. |
