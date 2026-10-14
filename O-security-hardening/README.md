# O. Security Hardening

> **Status:** 📝 Not yet tested in lab

## Concept

A MikroTik exposed to the internet gets scanned within minutes. Hardening means:
1. **Least exposure:** only the needed services, only from trusted networks.
2. **Strong authentication:** no default `admin`, strong passwords, SSH keys.
3. **Firewall first:** drop everything to the router that isn't explicitly allowed.
4. **Stay updated:** many past mass compromises hit routers running old RouterOS.

Apply in order, using **Safe Mode** (`Ctrl+X`).

## Checklist

### 1. Users

```
/user add name=fayez group=full password="Long-Unique-Passphrase!"
# log out, log back in as fayez, then:
/user disable admin
/user set fayez address=192.168.1.0/24,192.168.99.0/24    # allowed login source networks
```

### 2. Disable unused services

```
/ip service print
/ip service disable telnet,ftp,www,api,api-ssl
/ip service set ssh port=22 address=192.168.1.0/24,192.168.99.0/24
/ip service set winbox address=192.168.1.0/24,192.168.99.0/24
/ip service set www-ssl disabled=yes
```

Keep only what you use (usually **winbox** and **ssh**), always with `address=`. Changing ports (e.g. SSH to 2222) reduces log noise but **isn't** real security.

### 3. Strong SSH

```
/ip ssh set strong-crypto=yes host-key-type=ed25519
/ip ssh regenerate-host-key
```

Add SSH key login (see [B. Basics](../B-basics/)) and optionally turn off password login for SSH: `/ip ssh set always-allow-password-login=no`.

### 4. Restrict MAC access and discovery to the LAN

```
/tool mac-server set allowed-interface-list=LAN
/tool mac-server mac-winbox set allowed-interface-list=LAN
/tool mac-server ping set enabled=no
/ip neighbor discovery-settings set discover-interface-list=LAN
```

### 5. Turn off features you don't use

```
/tool bandwidth-server set enabled=no
/ip proxy set enabled=no
/ip socks set enabled=no
/ip upnp set enabled=no
/ip cloud set ddns-enabled=no update-time=no
/tool romon set enabled=no
/ip smb set enabled=no
/ip dns set allow-remote-requests=no      # only if the LAN doesn't use the router for DNS
/lcd set enabled=no                        # devices with an LCD
```

If the LAN uses the router for DNS, keep `allow-remote-requests=yes` and make sure the firewall input chain drops port 53 from WAN.

### 6. Firewall

Apply the base input and forward chains from [E. Firewall](../E-firewall/). The key rule is:

```
add chain=input action=drop in-interface-list=!LAN comment="IN: drop all not from LAN"
```

Add brute-force protection for any service you **must** expose (section 5 of E).

RAW drop for bogons arriving on WAN (spoofed private sources):

```
/ip firewall address-list
add list=BOGONS address=0.0.0.0/8
add list=BOGONS address=127.0.0.0/8
add list=BOGONS address=169.254.0.0/16
add list=BOGONS address=224.0.0.0/3
/ip firewall raw add chain=prerouting in-interface-list=WAN src-address-list=BOGONS action=drop comment="Drop bogons from WAN"
```

Don't add `10.0.0.0/8`, `172.16.0.0/12` or `192.168.0.0/16` to bogons if your WAN is behind a modem or CGNAT using those ranges. In this lab, the "WAN" is `192.168.1.0/24`.

### 7. IPv6

Secure it or disable it:

```
/ipv6 settings set disable-ipv6=yes
```

Otherwise, configure `/ipv6 firewall filter` the same way as IPv4.

### 8. Updates

```
/system package update set channel=stable
/system package update check-for-updates
/system package update install
```

Physical devices: `/system routerboard upgrade` after upgrading RouterOS.

Auto-check script example (log only):

```
/system scheduler add name=check-updates interval=1d on-event="/system package update check-for-updates; :delay 10s; :if ([/system package update get status] = \"New version is available\") do={ :log warning \"RouterOS update available\" }"
```

### 9. Device-mode (RouterOS 7.17+)

Device-mode locks risky features (packet sniffer, scheduler-based scripts, proxy, socks, etc.) and requires **physical** confirmation (a button press or power cycle) to unlock. This protects against a compromised admin account.

```
/system device-mode print
/system device-mode update mode=home      # or enterprise, depending on needs
```

Read the docs before changing device-mode. Some modes disable features you might use (scheduler, fetch).

### 10. Logging and audit

```
/system logging add topics=account action=memory      # logins and logouts
/log print where topics~"account"
/user active print
```

Send logs to remote syslog (see [M. Monitoring](../M-monitoring/)) so an attacker can't erase them.

### 11. Physical and backup hygiene

- Use an encrypted backup: `/system backup save password=...`
- Store exports **off** the router.
- Keep the reset button behavior in mind: anyone with physical access can reset the device. `/system routerboard settings set protected-routerboot=enabled` exists, but **understand it first**, because you can permanently lock yourself out.

## Quick audit commands

```
/ip service print
/user print
/user active print
/tool mac-server print
/ip neighbor discovery-settings print
/ip firewall filter print
/ip dns print
/ip proxy print
/ip socks print
/system package update print
/export compact
```

## EVE-NG lab

1. Create user `fayez`, disable `admin`, and verify login.
2. Restrict winbox and ssh to `192.168.1.0/24`. From your PC it still works.
3. Put another node on a different subnet (e.g. a VPCS behind R2) and try SSH to R1. It's blocked.
4. Disable the unused services and run the audit commands.
5. Port-scan R1 from a Linux node with `nmap -p- 192.168.1.133`, before and after hardening. Compare.

## Troubleshooting

| Problem | Fix |
|---|---|
| Locked out after `address=` restriction | Your current IP isn't in the list. Use the console (EVE-NG) or MAC WinBox from the LAN. |
| MAC WinBox stopped working | `allowed-interface-list` doesn't include the interface you connect from. |
| Can't log in after disabling admin | Your new user's group isn't `full`, or the password was mistyped. Use the console and reset. |
| Device-mode blocks a feature | Expected. Change the mode with physical confirmation. |
