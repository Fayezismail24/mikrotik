# M. Monitoring and Troubleshooting Tools

> **Status:** 📝 Not yet tested in lab
>
> **Cert:** MTCNA module 9 (Tools)

## Concept

RouterOS has built-in tools for:
- **Live troubleshooting:** ping, traceroute, torch, sniffer, bandwidth test.
- **History:** logs, graphs, traffic flow.
- **Alerts:** Netwatch, email, scripts.
- **Central monitoring:** SNMP, syslog, The Dude.

🖱️ **In WinBox,** most of these live under **Tools**: Ping, Traceroute, Torch, Packet Sniffer, Bandwidth Test, BTest Server, IP Scan, Netwatch, Email, Graphing, Profile.

---

## 1. Resources and health

| Check | 🖱️ WinBox | ⌨️ CLI |
|---|---|---|
| CPU, RAM, uptime, version | System → Resources | `/system resource print` |
| Per-core CPU | System → Resources → **CPU** button | `/system resource cpu print` |
| Temperature/voltage (hardware) | System → Health | `/system health print` |
| What uses the CPU | Tools → **Profile** → Start | `/tool profile duration=10s` |

---

## 2. Logs

### View

🖱️ **WinBox:** **Log** (left menu). Type in the filter box at the top-right to filter by topic or message.

⌨️ **CLI:**
```
/log print
/log print where topics~"error|critical"
/log print where message~"login"
/log print follow
```

### Remote syslog

🖱️ **WinBox:** System → **Logging**:
1. **Actions** tab → **+** → Name `syslog`, Type `remote`, Remote Address `192.168.1.108`, Remote Port `514` → OK.
2. **Rules** tab → **+** → Topics `info`, Action `syslog`. Repeat for `error` and `firewall`.

⌨️ **CLI:**
```
/system logging action add name=syslog target=remote remote=192.168.1.108 remote-port=514
/system logging add topics=info action=syslog
/system logging add topics=error action=syslog
/system logging add topics=firewall action=syslog
```

### Keep logs on disk (survive reboot)

🖱️ **WinBox:** System → Logging:
1. **Actions** → double-click `disk` → Lines Per File `1000`, File Count `5`.
2. **Rules** → **+** → Topics `critical`, Action `disk`.

⌨️ **CLI:**
```
/system logging action set disk disk-file-name=log disk-lines-per-file=1000 disk-file-count=5
/system logging add topics=critical action=disk
```

---

## 3. Live troubleshooting tools

### Ping

🖱️ **WinBox:** Tools → **Ping** → Ping To `8.8.8.8`, Count `5` (Advanced: Src. Address, Size, Don't Fragment) → Start

⌨️ **CLI:**
```
/ping 8.8.8.8 count=5
/ping 8.8.8.8 src-address=192.168.10.1 size=1472 do-not-fragment
```

### Traceroute

🖱️ **WinBox:** Tools → **Traceroute** → Address `8.8.8.8` → Start

⌨️ **CLI:**
```
/tool traceroute 8.8.8.8
```

### Torch (who uses bandwidth right now)

🖱️ **WinBox:** Tools → **Torch** → Interface `ether1`. Tick the columns you want (Src. Address, Dst. Address, Port) → Start.

⌨️ **CLI:**
```
/tool torch interface=ether1
```

### Packet sniffer

🖱️ **WinBox:** Tools → **Packet Sniffer**:
- **General** tab: File Name `cap.pcap`, File Limit `1000` KiB.
- **Filter** tab: Interface `ether1`, IP Address `192.168.10.10`.

Click **Start**, then **Stop**. Download `cap.pcap` from **Files** and open it in Wireshark.

⌨️ **CLI:**
```
/tool sniffer set filter-interface=ether1 filter-ip-address=192.168.10.10 file-name=cap.pcap file-limit=1000KiB
/tool sniffer start
/tool sniffer stop
```

**Live to Wireshark:** in Packet Sniffer → **Streaming** tab → Streaming Enabled ✓, Server `192.168.1.108`. In Wireshark, filter on `udp port 37008` and decode it as TZSP.

### Bandwidth test

🖱️ **WinBox:**
- Target router: Tools → **BTest Server** → Enabled ✓.
- Source router: Tools → **Bandwidth Test** → Test To `10.0.12.2`, Protocol `tcp`, Direction `both`, User/Password → Start.

⌨️ **CLI:**
```
/tool bandwidth-server set enabled=yes authenticate=yes
/tool bandwidth-test address=10.0.12.2 user=admin password=xxx direction=both protocol=tcp duration=15s
```

Disable the BTest server when you're done.

### IP scan and neighbors

| Task | 🖱️ WinBox | ⌨️ CLI |
|---|---|---|
| Scan a subnet | Tools → **IP Scan** → Interface `bridge1` → Start | `/tool ip-scan interface=bridge1` |
| Neighbors (MNDP/CDP/LLDP) | IP → **Neighbors** | `/ip neighbor print` |
| ARP table | IP → ARP | `/ip arp print` |

---

## 4. Netwatch (up/down actions)

🖱️ **WinBox:** Tools → **Netwatch** → **+**:
- Host `8.8.8.8`
- Type `icmp`
- Interval `00:00:10`
- Timeout `1s`
- **Up** tab: `:log info "INTERNET UP"`
- **Down** tab: `:log warning "INTERNET DOWN"`

⌨️ **CLI:**
```
/tool netwatch add host=8.8.8.8 interval=10s timeout=1s type=icmp \
  down-script=":log warning \"INTERNET DOWN\"" \
  up-script=":log info \"INTERNET UP\""
```

### Email alerts

🖱️ **WinBox:** Tools → **Email**:
- Server `smtp.gmail.com`
- Port `587`
- TLS `starttls`
- User and Password: use a Gmail **app password**
- From: your address

Then in the Netwatch **Down** tab:
```
/tool e-mail send to=you@gmail.com subject="R1: Internet down"
```

⌨️ **CLI:**
```
/tool e-mail set server=smtp.gmail.com port=587 tls=starttls user=you@gmail.com password="<app-password>" from=you@gmail.com
/tool netwatch set [find host=8.8.8.8] down-script="/tool e-mail send to=you@gmail.com subject=\"R1: Internet down\""
```

---

## 5. Graphs

🖱️ **WinBox:** Tools → **Graphing**:
- **Interface Rules** → **+** → Interface `ether1`, Allow Address `192.168.1.0/24`.
- **Resource Rules** → **+** → Allow Address `192.168.1.0/24`.
- **Queue Rules** → **+** → Allow Address `192.168.1.0/24`.

View them at `http://router-ip/graphs/` (the www service must be enabled).

⌨️ **CLI:**
```
/tool graphing interface add interface=ether1 allow-address=192.168.1.0/24
/tool graphing resource add allow-address=192.168.1.0/24
/tool graphing queue add allow-address=192.168.1.0/24
```

---

## 6. SNMP

🖱️ **WinBox:** IP → **SNMP**:
- Enabled ✓, Contact `Fayez`, Location `EVE-NG lab`.
- **Communities** button → **+** → Name `labmon`, Addresses `192.168.1.0/24`, Read Access ✓, Write Access ✗.
- Disable the `public` community.

⌨️ **CLI:**
```
/snmp community add name=labmon addresses=192.168.1.0/24 read-access=yes write-access=no
/snmp community set [find name=public] disabled=yes
/snmp set enabled=yes contact="Fayez" location="EVE-NG lab" trap-version=2
```

**SNMPv3:** in the community dialog, set Security `private`, Authentication Protocol `SHA1` plus password, and Encryption Protocol `AES` plus password.

```
/snmp community add name=v3user security=private authentication-protocol=SHA1 authentication-password=AuthPass123 encryption-protocol=AES encryption-password=EncPass123 addresses=192.168.1.0/24
```

---

## 7. Traffic flow (NetFlow / IPFIX)

🖱️ **WinBox:** IP → **Traffic Flow**:
- Enabled ✓, Interfaces `ether1`.
- **Targets** button → **+** → Dst. Address `192.168.1.108`, Port `2055`, Version `9`.

⌨️ **CLI:**
```
/ip traffic-flow set enabled=yes interfaces=ether1
/ip traffic-flow target add dst-address=192.168.1.108 port=2055 version=9
```

---

## 8. The Dude

The Dude is MikroTik's free network map and monitoring tool. The server is the `dude` package (works on CHR); the client is a Windows app.

### Install

🖱️ **WinBox:**
1. Download `dude-7.x.x.npk` (x86, the **same version** as RouterOS).
2. Drag it into **Files**, then **System → Reboot**.
3. A **Dude** menu appears → Settings → Enabled ✓, Data Directory `dude`.
4. Open the Dude client, connect to the router IP, and run **Discover**.

⌨️ **CLI:**
```
/dude set enabled=yes data-directory=dude
```

---

## 9. supout.rif (support output file)

### What is it?

A **supout.rif** is a full diagnostic snapshot of the router: config, logs, routing tables, interface stats, CPU and memory state, and more. It's packed into one file that **MikroTik support** reads with their viewer.

**Analogy:** a doctor's full blood test. You don't read it yourself; you hand it to the specialist.

### supout.rif vs autosupout.rif vs backup vs export

| File | Made when | For | Readable by you? |
|---|---|---|---|
| **supout.rif** | You create it manually | Sending to MikroTik support for a problem happening **now** | Only with the viewer in your mikrotik.com account |
| **autosupout.rif** | Automatically, after a **crash** (watchdog reboot) | Showing support what the router looked like when it crashed | Same |
| `.backup` | You create it | Restoring config | ❌ |
| `.rsc` export | You create it | Reading and sharing config | ✅ |

> **Privacy:** a supout contains your full config, including addresses and possibly secrets. Only send it to MikroTik support, and never post it publicly.

### Create a supout.rif

🖱️ **WinBox:** **Make Supout.rif** in the left menu (on some versions it's under Files). Wait until it finishes, then drag `supout.rif` from **Files** to your PC.

⌨️ **CLI:**
```
/system sup-output name=supout.rif
/file print where name~"supout"
```

### Auto-supout on crash

🖱️ **WinBox:** System → **Watchdog** → Automatic Supout ✓

⌨️ **CLI:**
```
/system watchdog set automatic-supout=yes
```

### Send to support

1. Log in at mikrotik.com → **Support** → you can open a supout in the **Supout viewer** yourself.
2. Email support@mikrotik.com with a description of the problem and the `supout.rif` attached, made **while the problem is happening**.

---

## EVE-NG lab

1. Netwatch on 8.8.8.8. Disconnect the Cloud0 link in EVE-NG and watch the Log.
2. Sniff a ping from PC1, download the pcap, and open it in Wireshark.
3. Remote syslog to your PC (`192.168.1.108`).
4. Install The Dude on a separate CHR and discover the lab.
5. SNMP: `snmpwalk -v2c -c labmon 192.168.1.133` from your PC.

## Troubleshooting

| Problem | Fix |
|---|---|
| Syslog server gets nothing | Windows firewall blocks UDP 514. |
| SNMP timeout | The community's Addresses doesn't include your PC, or the firewall drops UDP 161. |
| Graphs page 403 | Allow Address doesn't include your PC, or www is disabled. |
| Netwatch flapping | Raise the timeout and interval. |
| Dude package won't load | Version or architecture mismatch. |
