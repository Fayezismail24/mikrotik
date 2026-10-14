# L. Hotspot and PPPoE

> **Status:** 📝 Not yet tested in lab

## Concept

| | Hotspot | PPPoE |
|---|---|---|
| What | Captive portal (login page in the browser) | Username/password tunnel over Ethernet |
| Used in | Cafés, hotels, guest Wi-Fi | ISPs and WISPs |
| Client needs | A browser | A PPPoE client (router or OS dialer) |
| Limits via | Hotspot user profiles | PPP profiles |
| Central auth | RADIUS / User Manager | RADIUS / User Manager |

🖱️ **In WinBox:**
- **IP → Hotspot** has the tabs Servers, Server Profiles, Users, User Profiles, Active, Hosts, IP Bindings, Walled Garden, Walled Garden IP List.
- **PPP** has the tabs Interface, PPPoE Servers, Secrets, Profiles, Active Connections.

---

## Part 1: Hotspot

### Setup wizard

🖱️ **WinBox:** IP → Hotspot → **Servers** tab → **Hotspot Setup** → answer the prompts:

| Prompt | Lab value |
|---|---|
| HotSpot Interface | `ether2` |
| Local Address of Network | `192.168.50.1/24` |
| Masquerade Network | ✓ |
| Address Pool of Network | `192.168.50.10-192.168.50.254` |
| Select Certificate | `none` |
| IP Address of SMTP Server | `0.0.0.0` |
| DNS Servers | `1.1.1.1` |
| DNS Name | `hotspot.lab` |
| Name / Password of Local HotSpot User | `admin` / your choice |

⌨️ **CLI:**
```
/ip hotspot setup
```

The wizard creates the IP, pool, DHCP server, hotspot server, profile, NAT and firewall rules.

### User profiles

🖱️ **WinBox:** IP → Hotspot → **User Profiles** → **+**:
- `1hour`: Rate Limit `2M/5M`, Session Timeout `01:00:00`, Shared Users `1`.
- `vip`: Rate Limit `10M/20M`, Shared Users `3`.

⌨️ **CLI:**
```
/ip hotspot user profile add name=1hour rate-limit=2M/5M session-timeout=1h shared-users=1
/ip hotspot user profile add name=vip rate-limit=10M/20M shared-users=3
```

### Users

🖱️ **WinBox:** IP → Hotspot → **Users** → **+**:
- Name `guest1`, Password `1234`, Profile `1hour`.
- For a data cap, open the **Limits** tab and set Limit Bytes Total.

⌨️ **CLI:**
```
/ip hotspot user add name=guest1 password=1234 profile=1hour
/ip hotspot user add name=boss password=Secret! profile=vip limit-bytes-total=5G
```

### Walled garden (allowed without login)

🖱️ **WinBox:**
- IP → Hotspot → **Walled Garden** → **+** → Dst. Host `*.mycompany.com` → OK.
- **Walled Garden IP List** → **+** → Dst. Address `192.168.50.5`, Action `accept` → OK.

⌨️ **CLI:**
```
/ip hotspot walled-garden add dst-host=*.mycompany.com
/ip hotspot walled-garden ip add dst-address=192.168.50.5 action=accept
```

### Bypass a device (printer, camera)

🖱️ **WinBox:** IP → Hotspot → **IP Bindings** → **+** → MAC Address `AA:BB:CC:DD:EE:FF`, Type `bypassed` → OK

⌨️ **CLI:**
```
/ip hotspot ip-binding add mac-address=AA:BB:CC:DD:EE:FF type=bypassed comment="Printer"
```

### Customize the login page

🖱️ **WinBox:** **Files** → `hotspot/login.html` → drag it to your PC, edit it, and drag it back.

⌨️ **CLI:** no editor; use WinBox Files or `scp`.

### Monitor and kick users

| Task | 🖱️ WinBox | ⌨️ CLI |
|---|---|---|
| Logged-in users | IP → Hotspot → **Active** | `/ip hotspot active print` |
| All seen devices | IP → Hotspot → **Hosts** | `/ip hotspot host print` |
| Kick a user | Active → select → **−** | `/ip hotspot active remove [find user=guest1]` |

### User Manager (RADIUS, vouchers)

User Manager is an extra package (`user-manager`), managed in **WebFig** at `http://router-ip/um`.

🖱️ **WinBox:**
1. **RADIUS** → **+** → Service `hotspot`, Address `127.0.0.1`, Secret `RadSecret`.
2. IP → Hotspot → **Server Profiles** → double-click → **RADIUS** tab → Use RADIUS ✓.

⌨️ **CLI:**
```
/radius add service=hotspot address=127.0.0.1 secret=RadSecret
/ip hotspot profile set [find] use-radius=yes
/user-manager set enabled=yes
/user-manager router add name=local address=127.0.0.1 shared-secret=RadSecret
```

---

## Part 2: PPPoE server (ISP side)

### Step 1: IP pool

🖱️ **WinBox:** IP → Pool → **+** → `pppoe-pool`, `10.100.0.10-10.100.0.254`

⌨️ **CLI:**
```
/ip pool add name=pppoe-pool ranges=10.100.0.10-10.100.0.254
```

### Step 2: profiles (plans)

🖱️ **WinBox:** PPP → **Profiles** → **+**:
- **General** tab:
  - Name `plan-10M`
  - Local Address `10.100.0.1`
  - Remote Address `pppoe-pool`
  - DNS Server `1.1.1.1`
- **Limits** tab:
  - Rate Limit `5M/10M`
  - Only One `yes`

Repeat for `plan-50M` (`20M/50M`).

⌨️ **CLI:**
```
/ppp profile add name=plan-10M local-address=10.100.0.1 remote-address=pppoe-pool dns-server=1.1.1.1,8.8.8.8 rate-limit=5M/10M only-one=yes
/ppp profile add name=plan-50M local-address=10.100.0.1 remote-address=pppoe-pool dns-server=1.1.1.1,8.8.8.8 rate-limit=20M/50M only-one=yes
```

### Step 3: PPPoE server

🖱️ **WinBox:** PPP → **PPPoE Servers** → **+**:
- Service Name `ISP`
- Interface `ether2`
- Default Profile `plan-10M`
- One Session Per Host ✓
- Authentication: `pap`, `chap`, `mschap2`

⌨️ **CLI:**
```
/interface pppoe-server server add service-name=ISP interface=ether2 default-profile=plan-10M authentication=pap,chap,mschap2 one-session-per-host=yes disabled=no
```

The PPPoE interface (`ether2`) **doesn't need an IP**.

### Step 4: customers

🖱️ **WinBox:** PPP → **Secrets** → **+** → Name `customer1`, Password, Service `pppoe`, Profile `plan-10M`. For a static IP, set Remote Address.

⌨️ **CLI:**
```
/ppp secret add name=customer1 password=Cust1Pass service=pppoe profile=plan-10M
/ppp secret add name=customer2 password=Cust2Pass service=pppoe profile=plan-50M remote-address=10.100.0.50
```

### Step 5: NAT for customers

🖱️ **WinBox:** IP → Firewall → NAT → **+** → Chain `srcnat`, Src. Address `10.100.0.0/24`, Out. Interface List `WAN`, Action `masquerade`

⌨️ **CLI:**
```
/ip firewall nat add chain=srcnat src-address=10.100.0.0/24 out-interface-list=WAN action=masquerade
```

### MTU / MSS (prevents broken websites)

🖱️ **WinBox:**
- PPP → Profiles → each profile → **Protocols** tab → Change TCP MSS `yes`.
- PPPoE Servers → Max MTU / Max MRU `1480`.

⌨️ **CLI:**
```
/ppp profile set [find] change-tcp-mss=yes
/interface pppoe-server server set [find] max-mtu=1480 max-mru=1480
```

### Monitor

| Task | 🖱️ WinBox | ⌨️ CLI |
|---|---|---|
| Online customers | PPP → **Active Connections** | `/ppp active print` |
| Per-user queues | Queues → Simple Queues (dynamic) | `/queue simple print` |
| Disconnect | Active Connections → select → **−** | `/ppp active remove [find name=customer1]` |

---

## Part 3: PPPoE client

🖱️ **WinBox:** PPP → **Interface** → **+** → **PPPoE Client**:
- General tab: Name `pppoe-out1`, Interfaces `ether1`.
- **Dial Out** tab:
  - User `customer1`
  - Password
  - Use Peer DNS ✓
  - Add Default Route ✓

Click **OK**. Then go to Interfaces → Interface List → **+** → List `WAN`, Interface `pppoe-out1`.

⌨️ **CLI:**
```
/interface pppoe-client add name=pppoe-out1 interface=ether1 user=customer1 password=Cust1Pass add-default-route=yes use-peer-dns=yes disabled=no
/interface list member add list=WAN interface=pppoe-out1
/interface pppoe-client monitor pppoe-out1
```

### ISP requires a VLAN (common with fiber)

🖱️ **WinBox:**
1. Interfaces → **+ VLAN** → `vlan35-wan`, VLAN ID `35`, Interface `ether1`.
2. Set the PPPoE client's Interfaces to `vlan35-wan`.

⌨️ **CLI:**
```
/interface vlan add interface=ether1 name=vlan35-wan vlan-id=35
/interface pppoe-client set pppoe-out1 interface=vlan35-wan
```

---

## EVE-NG lab

### Hotspot

```
[Cloud0] --ether1-- R1 --ether2-- client node with a browser (Linux/Windows VM)
```

1. Run Hotspot Setup on ether2.
2. Add `guest1` with profile `1hour`.
3. In the client browser, open `http://neverssl.com`. You get redirected to the login page. Log in.
4. Check IP → Hotspot → Active.

### PPPoE

```
[Cloud0] --ether1-- ISP-R1 --ether2-- CUST-R2 --ether2-- PC1
```

1. On ISP-R1: pool, profile, server, secrets, NAT.
2. On CUST-R2: PPPoE client on ether1, then LAN + DHCP + masquerade on ether2.
3. PC1 pings 8.8.8.8.
4. On ISP-R1: PPP → Active Connections shows `customer1`.

## Troubleshooting

| Problem | Fix |
|---|---|
| No login page | HTTPS-only site (use `http://neverssl.com`), or DNS doesn't go through the router. |
| Can't browse after login | Missing NAT, or the hotspot is on the wrong interface. |
| PPPoE stuck connecting | Wrong service name or interface, or a VLAN is required. |
| PPPoE auth fails | Username/password or auth method mismatch. Check the Log (topic `ppp`). |
| Some sites don't load | MTU/MSS. Set Change TCP MSS to `yes`. |
