# O. Security Hardening

> **Status:** 📝 Not yet tested in lab
>
> **Cert:** MTCSE · basics appear in MTCNA module 1

## Concept

An internet-facing MikroTik gets scanned within minutes. Hardening means:
1. **Least exposure:** only the needed services, only from trusted networks.
2. **Strong authentication:** no default admin, strong passwords, SSH keys.
3. **Firewall first:** drop everything to the router that isn't allowed.
4. **Stay updated.**

Turn on **Safe Mode** before starting.

---

## 1. Users

🖱️ **WinBox:**
1. System → Users → **+** → your own user, Group `full`.
2. Test it in a new WinBox window, then disable `admin` (**✕**).
3. Double-click your user → Allowed Address `192.168.1.0/24` → OK.

⌨️ **CLI:**
```
/user add name=yourname group=full password="Long-Unique-Passphrase!"
/user disable admin
/user set yourname address=192.168.1.0/24
```

Step-by-step details are in [B. Basics § 6](../B-basics/).

---

## 2. Disable unused services

🖱️ **WinBox:** IP → **Services**:
- Select `telnet`, `ftp`, `www`, `api`, `api-ssl` → **✕** (Disable).
- Double-click `ssh` → Available From `192.168.1.0/24` → OK.
- Double-click `winbox` → Available From `192.168.1.0/24` → OK.

⌨️ **CLI:**
```
/ip service disable telnet,ftp,www,api,api-ssl
/ip service set ssh address=192.168.1.0/24
/ip service set winbox address=192.168.1.0/24
/ip service print
```

Changing ports (e.g. SSH to 2222) cuts log noise but **isn't** real security.

---

## 3. Strong SSH

🖱️ **WinBox:** IP → **SSH** → Strong Crypto ✓, Host Key Type `ed25519` → Apply → **Regenerate Host Key**

⌨️ **CLI:**
```
/ip ssh set strong-crypto=yes host-key-type=ed25519
/ip ssh regenerate-host-key
```

---

## 4. Restrict MAC access and discovery to the LAN

🖱️ **WinBox:**
- Tools → **MAC Server**:
  - **MAC Telnet Server**: Allowed Interface List `LAN`.
  - **MAC WinBox Server**: Allowed Interface List `LAN`.
  - **MAC Ping Server**: untick Enabled.
- IP → **Neighbors** → **Discovery Settings** → Interface List `LAN`.

⌨️ **CLI:**
```
/tool mac-server set allowed-interface-list=LAN
/tool mac-server mac-winbox set allowed-interface-list=LAN
/tool mac-server ping set enabled=no
/ip neighbor discovery-settings set discover-interface-list=LAN
```

---

## 5. Turn off features you don't use

| Feature | 🖱️ WinBox | ⌨️ CLI |
|---|---|---|
| Bandwidth test server | Tools → BTest Server → Enabled ✗ | `/tool bandwidth-server set enabled=no` |
| Web proxy | IP → Web Proxy → Enabled ✗ | `/ip proxy set enabled=no` |
| SOCKS | IP → SOCKS → Enabled ✗ | `/ip socks set enabled=no` |
| UPnP | IP → UPnP → Enabled ✗ | `/ip upnp set enabled=no` |
| Cloud DDNS | IP → Cloud → DDNS Enabled ✗ | `/ip cloud set ddns-enabled=no update-time=no` |
| RoMON | Tools → RoMON → Enabled ✗ | `/tool romon set enabled=no` |
| SMB | IP → SMB → Enabled ✗ | `/ip smb set enabled=no` |
| DNS server (if unused) | IP → DNS → Allow Remote Requests ✗ | `/ip dns set allow-remote-requests=no` |

If the LAN uses the router for DNS, keep it on, but make sure the firewall drops port 53 from WAN.

---

## 6. Firewall

Apply the base firewall from [E. Firewall](../E-firewall/). The key rule is:

```
add chain=input action=drop in-interface-list=!LAN comment="IN: drop all not from LAN"
```

### Drop bogons from WAN (RAW)

🖱️ **WinBox:**
1. IP → Firewall → **Address Lists** → **+** (four times), list `BOGONS`:
   - `0.0.0.0/8`
   - `127.0.0.0/8`
   - `169.254.0.0/16`
   - `224.0.0.0/3`
2. IP → Firewall → **Raw** → **+**:
   - General: Chain `prerouting`, In. Interface List `WAN`.
   - Advanced: Src. Address List `BOGONS`.
   - Action: `drop`.

⌨️ **CLI:**
```
/ip firewall address-list
add list=BOGONS address=0.0.0.0/8
add list=BOGONS address=127.0.0.0/8
add list=BOGONS address=169.254.0.0/16
add list=BOGONS address=224.0.0.0/3
/ip firewall raw add chain=prerouting in-interface-list=WAN src-address-list=BOGONS action=drop comment="Drop bogons from WAN"
```

Don't add `10/8`, `172.16/12` or `192.168/16` here. In this lab, the WAN **is** `192.168.1.0/24`.

---

## 7. IPv6

🖱️ **WinBox:** IPv6 → Settings → Disable IPv6 ✓ (or configure IPv6 → Firewall like IPv4)

⌨️ **CLI:**
```
/ipv6 settings set disable-ipv6=yes
```

---

## 8. Updates

🖱️ **WinBox:** System → Packages → **Check For Updates** → Channel `stable` → **Download&Install**

⌨️ **CLI:**
```
/system package update set channel=stable
/system package update check-for-updates
/system package update install
```

### Daily update check (log only)

🖱️ **WinBox:** System → Scheduler → **+** → Interval `1d`, On Event: the script below.

⌨️ **CLI:**
```
/system scheduler add name=check-updates interval=1d on-event="/system package update check-for-updates; :delay 10s; :if ([/system package update get status] = \"New version is available\") do={ :log warning \"RouterOS update available\" }"
```

---

## 9. Device-mode (RouterOS 7.17+)

Device-mode locks risky features and needs **physical** confirmation (a button press or power cycle) to unlock.

🖱️ **WinBox:** no menu for changing it. Use **New Terminal**.

⌨️ **CLI:**
```
/system device-mode print
/system device-mode update mode=home
```

Read the docs first. Some modes disable things like the scheduler or fetch.

---

## 10. Logging and audit

🖱️ **WinBox:**
1. System → Logging → **Rules** → **+** → Topics `account`, Action `memory`.
2. Review logins in **Log** (filter `account`) and System → Users → **Active Users**.

⌨️ **CLI:**
```
/system logging add topics=account action=memory
/log print where topics~"account"
/user active print
```

Send logs to remote syslog ([M](../M-monitoring/)) so an attacker can't erase them.

---

## 11. Backup hygiene

- Use an encrypted backup. 🖱️ Files → Backup → Password. ⌨️ `/system backup save password=...`
- Store exports **off** the router.
- `protected-routerboot` exists on hardware, but **understand it first**: you can permanently lock yourself out.

---

## Quick audit

| Check | 🖱️ WinBox | ⌨️ CLI |
|---|---|---|
| Services | IP → Services | `/ip service print` |
| Users | System → Users | `/user print` |
| Who is logged in | System → Users → Active Users | `/user active print` |
| MAC server | Tools → MAC Server | `/tool mac-server print` |
| Discovery | IP → Neighbors → Discovery Settings | `/ip neighbor discovery-settings print` |
| Firewall | IP → Firewall → Filter Rules | `/ip firewall filter print` |
| DNS | IP → DNS | `/ip dns print` |
| Updates | System → Packages → Check For Updates | `/system package update print` |
| Full config | (New Terminal) | `/export compact` |

## EVE-NG lab

1. Own user, admin disabled, verify the login.
2. Restrict winbox and ssh to `192.168.1.0/24`. It still works from your PC.
3. From a node on another subnet, try SSH to R1. It's blocked.
4. Disable the unused services and run the audit.
5. `nmap -p- 192.168.1.133` from a Linux node, before and after. Compare.

## Troubleshooting

| Problem | Fix |
|---|---|
| Locked out after the Available From change | Your IP isn't in the list. Use the EVE-NG console or MAC WinBox from the LAN. |
| MAC WinBox stopped working | The Allowed Interface List doesn't include your interface. |
| Can't log in after disabling admin | The new user isn't group `full`. Fix it from the console. |
| Device-mode blocks a feature | Expected. Change the mode with physical confirmation. |
