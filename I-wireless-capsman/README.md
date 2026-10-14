# I. Wireless and CAPsMAN

> **Status:** 📝 Not yet tested. Hardware available: a spare hAP bought for MTCNA (not the home router, free to reset; model to be confirmed). CHR has no wireless hardware, so the Wi-Fi radios in this section come from the hAP.
>
> **Cert:** MTCNA module 5 (Wireless) · MTCWE

## Concept

RouterOS 7 has **two wireless drivers**:

| Package | WinBox menu | CLI menu | Devices | CAPsMAN |
|---|---|---|---|---|
| **wifi** (new) | **WiFi** | `/interface wifi` | Wi-Fi 5/6 devices: hAP ax², ax³, cAP ax... | WiFi → **CAPsMAN** |
| **wireless** (legacy) | **Wireless** | `/interface wireless` | Older 802.11n/ac: hAP ac², hAP lite... | **CAPsMAN** menu |

### Which driver do I have?

🖱️ **WinBox:** System → Packages, and check whether **WiFi** or **Wireless** appears in the left menu.

⌨️ **CLI:**
```
/system package print
/interface print where type~"wifi|wlan"
```

### Which package does my hAP use?

It depends on the Wi-Fi chip, which depends on the model:

| hAP model | Wi-Fi generation | Package | WinBox menu |
|---|---|---|---|
| hAP ax lite, hAP ax², hAP ax³, hAP ax S | Wi-Fi 6 | **wifi** (new) | **WiFi** |
| hAP ac², hAP ac³, hAP ac lite | Wi-Fi 5 | **wireless** (legacy) | **Wireless** |
| hAP lite, hAP mini | Wi-Fi 4 (2.4 GHz only) | **wireless** (legacy) | **Wireless** |

Find the model and version:

🖱️ **WinBox:** System → **RouterBOARD** (model) and System → **Resources** (version, board name)

⌨️ **CLI:**
```
/system routerboard print
/system resource print
/system package print
```

**CAPsMAN** gives central management of many APs. The **controller** holds the config, and each **CAP** (access point) receives it.

---

## Before you start: if the hAP is your home router

If your family's internet and Wi-Fi run through this hAP, one wrong setting takes the house offline. Protect it first.

**1. Take a backup and an export, and download both to your PC** (see [B. Basics § 7](../B-basics/)):

🖱️ **WinBox:** Files → **Backup** (with a password). Then in New Terminal run `/export file=hap-before-lab`. Drag both files to your desktop.

⌨️ **CLI:**
```
/system backup save name=hap-before-lab password="YourPass"
/export file=hap-before-lab show-sensitive
```

**2. Use Safe Mode for every change.** 🖱️ the **Safe Mode** button / ⌨️ `Ctrl+X`. If you lose the connection, the changes roll back.

**3. Don't change the main SSID.** Add a separate **test SSID** (a virtual AP, see "Guest network" below) and experiment on that. The family keeps their normal Wi-Fi.

**4. Know your way back.**
- 🖱️ Files → select `hap-before-lab.backup` → **Restore**.
- ⌨️ `/system backup load name=hap-before-lab.backup`

If WinBox is unreachable, connect a cable to a LAN port and use **MAC WinBox** from the Neighbors tab.

---

## Lab plan: CHR controller + real hAP

```
EVE-NG (VMware)                         Home LAN 192.168.1.0/24
  R1 (CHR) ether1 --- Cloud0/pnet0 --- home switch/router --- hAP (CAP)
  192.168.1.133                                              192.168.1.x
```

- **R1 (CHR)** runs the **CAPsMAN controller**. CHR can be a controller; it just has no radios of its own.
- **The hAP** runs as a **CAP**. It finds R1 over the home LAN and gets its Wi-Fi config from it.
- Use the CAPsMAN type that matches the hAP's package: **wifi** CAPsMAN (Part 3) or legacy CAPsMAN (Part 4). The controller and CAP must use the same type.
- If the hAP is the main home router, enabling CAP mode hands its radios to R1, which **replaces** its current Wi-Fi config. Do this only after the backup above, and preferably on a spare device or a test SSID.

---

## Part 1: Standalone AP with `wifi` (new)

### Step 1: security profile

🖱️ **WinBox:** WiFi → **Security** tab → **+**:
- Name: `sec-home`
- Authentication Types: `WPA2 PSK`, `WPA3 PSK`
- Passphrase: `ChangeMe123!`

⌨️ **CLI:**
```
/interface wifi security add name=sec-home authentication-types=wpa2-psk,wpa3-psk passphrase="ChangeMe123!"
```

### Step 2: configuration

🖱️ **WinBox:** WiFi → **Configurations** tab → **+**:
- Name: `cfg-home`
- SSID: `HomeNet`
- Country: your country
- **Security** section: `sec-home`

⌨️ **CLI:**
```
/interface wifi configuration add name=cfg-home ssid="HomeNet" country="Lebanon" security=sec-home
```

### Step 3: apply to the radios and bridge them

🖱️ **WinBox:**
1. WiFi → **WiFi** tab → double-click `wifi1` → Configuration: `cfg-home` → OK → **Enable** (✓). Repeat for `wifi2`.
2. Bridge → **Ports** → **+** → `wifi1` on `bridge1`. Repeat for `wifi2`.

⌨️ **CLI:**
```
/interface wifi set wifi1 configuration=cfg-home disabled=no
/interface wifi set wifi2 configuration=cfg-home disabled=no
/interface bridge port add bridge=bridge1 interface=wifi1
/interface bridge port add bridge=bridge1 interface=wifi2
```

### Channel settings

🖱️ **WinBox:**
1. WiFi → **Channels** tab → **+** → Name `ch-5g`, Band `5ghz-ax`, Width `20/40/80mhz` → OK.
2. Configurations → `cfg-home` → Channel: `ch-5g`.

⌨️ **CLI:**
```
/interface wifi channel add name=ch-5g band=5ghz-ax width=20/40/80mhz
/interface wifi configuration set cfg-home channel=ch-5g
```

### Guest network (virtual AP on VLAN 30)

🖱️ **WinBox:**
1. Security → **+** `sec-guest` (WPA2 PSK, passphrase).
2. Configurations → **+** `cfg-guest` (SSID `Guest`, security `sec-guest`).
3. WiFi → **+** → Master Interface `wifi2`, Name `wifi2-guest`, Configuration `cfg-guest`.
4. Bridge → Ports → **+** → `wifi2-guest` on `bridge1`, VLAN tab PVID `30`.

⌨️ **CLI:**
```
/interface wifi security add name=sec-guest authentication-types=wpa2-psk passphrase="GuestPass123"
/interface wifi configuration add name=cfg-guest ssid="Guest" security=sec-guest
/interface wifi add master-interface=wifi2 name=wifi2-guest configuration=cfg-guest disabled=no
/interface bridge port add bridge=bridge1 interface=wifi2-guest pvid=30
```

Then add VLAN 30 to the bridge VLAN table ([D](../D-bridging-vlans/)) and isolate it in the firewall.

### Monitor

| Check | 🖱️ WinBox | ⌨️ CLI |
|---|---|---|
| Connected clients | WiFi → **Registration** tab | `/interface wifi registration-table print` |
| Radio status | WiFi → double-click → **Status** | `/interface wifi monitor wifi1` |
| Scan the air | WiFi → **Scan** button | `/interface wifi scan wifi1` |

---

## Part 2: Standalone AP with legacy `wireless`

### Security profile

🖱️ **WinBox:** Wireless → **Security Profiles** → **+**:
- Name: `home`
- Mode: `dynamic keys`
- WPA2 PSK: ✓
- WPA2 Pre-Shared Key: `ChangeMe123!`

⌨️ **CLI:**
```
/interface wireless security-profiles add name=home mode=dynamic-keys authentication-types=wpa2-psk wpa2-pre-shared-key="ChangeMe123!"
```

### Radio

🖱️ **WinBox:** Wireless → double-click `wlan1` → **Wireless** tab:
- Mode: `ap bridge`
- Band: `2GHz-B/G/N`
- SSID: `HomeNet`
- Frequency: `auto`
- Country: yours
- Security Profile: `home`

Click OK, then **Enable**.

⌨️ **CLI:**
```
/interface wireless set wlan1 mode=ap-bridge ssid="HomeNet" band=2ghz-b/g/n frequency=auto country=lebanon security-profile=home disabled=no
/interface wireless registration-table print
```

---

## Part 3: CAPsMAN with `wifi` (new)

### On the controller

🖱️ **WinBox:**
1. WiFi → **Security** → **+** `sec1`.
2. WiFi → **Configurations** → **+** `cfg1`:
   - SSID `Office`
   - Country
   - Security `sec1`
   - **Datapath** section: Bridge `bridge1`
3. WiFi → **CAPsMAN** button → Enabled ✓, Interfaces `bridge1` → OK.
4. WiFi → **Provisioning** tab → **+**:
   - Action `create dynamic enabled`, Master Configuration `cfg1`, Supported Bands `2ghz-ax`.
   - Add another for `5ghz-ax`.

⌨️ **CLI:**
```
/interface wifi security add name=sec1 authentication-types=wpa2-psk,wpa3-psk passphrase="ChangeMe123!"
/interface wifi configuration add name=cfg1 ssid="Office" country="Lebanon" security=sec1 datapath.bridge=bridge1
/interface wifi capsman set enabled=yes interfaces=bridge1 upgrade-policy=suggest-same-version
/interface wifi provisioning add action=create-dynamic-enabled master-configuration=cfg1 supported-bands=2ghz-ax
/interface wifi provisioning add action=create-dynamic-enabled master-configuration=cfg1 supported-bands=5ghz-ax
```

### On each CAP

🖱️ **WinBox:**
1. Bridge `bridge1` with `ether1` as a port. IP → DHCP Client on `bridge1`.
2. WiFi → **CAP** button → Enabled ✓, Discovery Interfaces `bridge1`, Slaves Datapath `bridge1` → OK.
3. WiFi → double-click each radio → Configuration → Manager `capsman`.

⌨️ **CLI:**
```
/interface bridge add name=bridge1
/interface bridge port add bridge=bridge1 interface=ether1
/ip dhcp-client add interface=bridge1 disabled=no
/interface wifi cap set enabled=yes discovery-interfaces=bridge1 slaves-datapath=bridge1
/interface wifi set [find] configuration.manager=capsman
```

The CAP-side syntax varies slightly between v7 releases. Check `/interface wifi print detail` after enabling it.

### Monitor

| Check | 🖱️ WinBox | ⌨️ CLI |
|---|---|---|
| Connected CAPs | WiFi → **Remote CAP** tab | `/interface wifi capsman remote-cap print` |
| Clients | WiFi → Registration | `/interface wifi registration-table print` |

---

## Part 4: Legacy CAPsMAN

### Controller

🖱️ **WinBox:** **CAPsMAN** menu:
1. **Security Cfg.** → **+** `sec1`.
2. **Configurations** → **+** `cfg1`:
   - Wireless tab: SSID `Office`, Country.
   - Datapath tab: Bridge `bridge1`.
   - Security tab: `sec1`.
3. **Manager** button → Enabled ✓.
4. **Provisioning** → **+** → Action `create dynamic enabled`, Master Configuration `cfg1`.

⌨️ **CLI:**
```
/caps-man security add name=sec1 authentication-types=wpa2-psk passphrase="ChangeMe123!"
/caps-man configuration add name=cfg1 ssid="Office" country=lebanon security=sec1 datapath.bridge=bridge1
/caps-man manager set enabled=yes
/caps-man provisioning add action=create-dynamic-enabled master-configuration=cfg1
```

### CAP

🖱️ **WinBox:** Wireless → **CAP** button:
- Enabled ✓
- Interfaces `wlan1`, `wlan2`
- Discovery Interfaces `bridge1`
- Bridge `bridge1`

⌨️ **CLI:**
```
/interface wireless cap set enabled=yes interfaces=wlan1,wlan2 discovery-interfaces=bridge1 bridge=bridge1
```

> The new and legacy CAPsMAN **can't** manage each other's CAPs.

---

## Practice without hardware

Get a cheap device (hAP lite, hAP ax lite) and connect it to the EVE-NG lab through Cloud0.

## Troubleshooting

| Problem | Fix |
|---|---|
| SSID not visible | Interface disabled, wrong country, or a DFS channel still waiting (5 GHz takes about 1 minute). |
| Clients connect but get no IP | Wi-Fi interface not in the bridge, or the datapath bridge isn't set. |
| CAP missing in CAPsMAN | Different L2 (use CAPsMAN addresses for L3), firewall blocks UDP 5246/5247, or a version mismatch. |
| Poor speed | Wrong width, interference (use Scan), or too much TX power. |
