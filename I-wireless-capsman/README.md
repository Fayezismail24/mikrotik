# I. Wireless and CAPsMAN

> **Status:** 📝 Not yet tested. **CHR has no wireless hardware**, so this section needs a physical MikroTik (hAP, cAP, etc.) to practice.

## Concept

RouterOS 7 has **two wireless drivers**. Which one you use depends on the device:

| Package | Menu | Devices | CAPsMAN |
|---|---|---|---|
| **wifi** (new, also called "wifiwave2") | `/interface wifi` | Wi-Fi 5/6 devices: hAP ax², ax³, cAP ax, Audience, etc. | `/interface wifi capsman` |
| **wireless** (legacy) | `/interface wireless` | Older 802.11n/ac: hAP ac², cAP ac, hAP lite, etc. | `/caps-man` |

Check which you have:

```
/system package print
/interface print where type~"wifi|wlan"
```

**CAPsMAN (Controlled Access Point system Manager)** is central management of many APs from one controller:
- The **CAPsMAN** is the controller that holds the configuration.
- **CAP** is an AP that gets its config from CAPsMAN.

Key terms:
- **SSID:** network name.
- **Band / channel width:** 2.4 GHz (range) vs 5 GHz (speed), at 20/40/80 MHz.
- **Security:** WPA2-PSK, WPA3-PSK, WPA2/3-Enterprise (RADIUS).
- **Country:** sets the legal channels and power. Always set it correctly.

---

## Part 1: Standalone AP with the new `wifi` package

```
/interface wifi security add name=sec-home authentication-types=wpa2-psk,wpa3-psk passphrase="ChangeMe123!"
/interface wifi configuration add name=cfg-home ssid="HomeNet" country="Lebanon" security=sec-home
/interface wifi set wifi1 configuration=cfg-home disabled=no
/interface wifi set wifi2 configuration=cfg-home disabled=no
/interface bridge port add bridge=bridge1 interface=wifi1
/interface bridge port add bridge=bridge1 interface=wifi2
```

Use your own country name. Tab-complete it after `country=`.

Channel settings:

```
/interface wifi channel add name=ch-5g band=5ghz-ax width=20/40/80mhz
/interface wifi configuration set cfg-home channel=ch-5g
```

### Guest network (virtual AP, own VLAN)

```
/interface wifi security add name=sec-guest authentication-types=wpa2-psk passphrase="GuestPass123"
/interface wifi configuration add name=cfg-guest ssid="Guest" security=sec-guest
/interface wifi add master-interface=wifi2 name=wifi2-guest configuration=cfg-guest disabled=no
/interface bridge port add bridge=bridge1 interface=wifi2-guest pvid=30
```

Then add VLAN 30 to the bridge VLAN table (see [D. Bridging and VLANs](../D-bridging-vlans/)), and isolate it in the firewall.

### Monitor

```
/interface wifi registration-table print
/interface wifi monitor wifi1
/interface wifi radio print
```

---

## Part 2: Standalone AP with legacy `wireless`

```
/interface wireless security-profiles add name=home mode=dynamic-keys authentication-types=wpa2-psk wpa2-pre-shared-key="ChangeMe123!"
/interface wireless set wlan1 mode=ap-bridge ssid="HomeNet" band=2ghz-g/n frequency=auto country=lebanon security-profile=home disabled=no
/interface wireless set wlan2 mode=ap-bridge ssid="HomeNet" band=5ghz-a/n/ac channel-width=20/40/80mhz-XXXX country=lebanon security-profile=home disabled=no
/interface wireless registration-table print
```

---

## Part 3: CAPsMAN with the new `wifi` package

### On the controller (usually the main router)

```
/interface wifi security add name=sec1 authentication-types=wpa2-psk,wpa3-psk passphrase="ChangeMe123!"
/interface wifi configuration add name=cfg1 ssid="Office" country="Lebanon" security=sec1 datapath.bridge=bridge1
/interface wifi capsman set enabled=yes interfaces=bridge1 upgrade-policy=suggest-same-version
/interface wifi provisioning add action=create-dynamic-enabled master-configuration=cfg1 supported-bands=2ghz-ax
/interface wifi provisioning add action=create-dynamic-enabled master-configuration=cfg1 supported-bands=5ghz-ax
```

If the controller's own radios should also be managed as CAPs, enable `/interface wifi cap set enabled=yes caps-man-addresses=127.0.0.1` on it. That's why the base firewall accepts `dst-address=127.0.0.1`.

### On each CAP (access point)

```
/interface bridge add name=bridge1
/interface bridge port add bridge=bridge1 interface=ether1
/ip dhcp-client add interface=bridge1 disabled=no
/interface wifi cap set enabled=yes discovery-interfaces=bridge1 slaves-datapath=bridge1
/interface wifi set [find] configuration.manager=capsman
```

The radio-to-CAPsMAN assignment syntax differs slightly between v7 minor releases. Check `/interface wifi print detail` on the CAP after enabling it.

### Monitor CAPsMAN

```
/interface wifi capsman remote-cap print
/interface wifi print
/interface wifi registration-table print
```

---

## Part 4: Legacy CAPsMAN (`/caps-man`)

Controller:

```
/caps-man security add name=sec1 authentication-types=wpa2-psk passphrase="ChangeMe123!"
/caps-man configuration add name=cfg1 ssid="Office" country=lebanon security=sec1 datapath.bridge=bridge1
/caps-man manager set enabled=yes
/caps-man provisioning add action=create-dynamic-enabled master-configuration=cfg1
```

CAP:

```
/interface wireless cap set enabled=yes interfaces=wlan1,wlan2 discovery-interfaces=bridge1 bridge=bridge1
```

Monitor: `/caps-man remote-cap print`, `/caps-man registration-table print`.

> The new `wifi` CAPsMAN **can't** manage legacy `wireless` CAPs, and vice versa. Keep one type per network, or run both controllers.

---

## Practice without hardware

- Study the config in WebFig/WinBox on a CHR. The menus exist only when the package is installed, so mostly you can't.
- Use a cheap device such as a hAP lite or hAP ax lite, connected to your EVE-NG lab through Cloud0.

## Troubleshooting

| Problem | Fix |
|---|---|
| SSID not visible | Interface disabled, wrong country (channel not allowed), or DFS channel waiting (5 GHz takes about 1 minute). |
| Clients connect but get no IP | Wi-Fi interface not in the bridge, or the datapath bridge isn't set. |
| CAP not showing in CAPsMAN | Not the same L2 network (use `caps-man-addresses=` for L3), firewall blocks UDP 5246/5247, or a version mismatch. |
| Poor speed | Wrong channel width, interference (`/interface wifi scan wifi1`), or too-high TX power causing roaming issues. |
| Mixed package confusion | Check `/system package print`. Use the menu that matches the installed driver. |
