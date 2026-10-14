# M. Monitoring and Troubleshooting Tools

> **Status:** 📝 Not yet tested in lab

## Concept

You can't fix what you can't see. RouterOS has built-in tools for:
- **Live troubleshooting:** ping, traceroute, torch, sniffer, bandwidth test.
- **History:** logs, graphs, traffic-flow (NetFlow).
- **Alerting and automation:** Netwatch, email, scripts.
- **Central monitoring:** SNMP, remote syslog, The Dude.

---

## 1. Resources and health

```
/system resource print          # CPU, RAM, uptime, version
/system resource cpu print      # per-core load
/system health print            # voltage/temperature (physical devices)
/tool profile duration=10s      # what is using the CPU
```

## 2. Logs

```
/log print
/log print where topics~"error|critical"
/log print where message~"login"
/log print follow               # live, Ctrl+C to stop
```

### Remote syslog

```
/system logging action add name=syslog target=remote remote=192.168.1.108 remote-port=514
/system logging add topics=info action=syslog
/system logging add topics=error action=syslog
/system logging add topics=firewall action=syslog
```

On Windows, a free syslog server such as Visual Syslog or Kiwi Syslog can receive these.

### Keep logs across reboots

```
/system logging action set disk disk-file-name=log disk-lines-per-file=1000 disk-file-count=5
/system logging add topics=critical action=disk
```

## 3. Live troubleshooting tools

### Ping and traceroute

```
/ping 8.8.8.8 count=5
/ping 8.8.8.8 src-address=192.168.10.1 size=1472 do-not-fragment     # MTU test
/tool traceroute 8.8.8.8
```

### Torch (who is using bandwidth right now)

```
/tool torch interface=ether1
/tool torch interface=bridge1 src-address=0.0.0.0/0 port=any
```

In WinBox, Tools → Torch is the easiest view.

### Packet sniffer

```
/tool sniffer set filter-interface=ether1 filter-ip-address=192.168.10.10 file-name=cap.pcap file-limit=1000KiB
/tool sniffer start
/tool sniffer stop
```

Download `cap.pcap` from Files and open it in **Wireshark**.

Stream live to Wireshark (TZSP):

```
/tool sniffer set streaming-enabled=yes streaming-server=192.168.1.108
```

In Wireshark on the PC, capture with the filter `udp port 37008` and decode as TZSP.

### Bandwidth test (router to router)

```
/tool bandwidth-server set enabled=yes authenticate=yes     # on the target
/tool bandwidth-test address=10.0.12.2 user=admin password=xxx direction=both protocol=tcp duration=15s
```

Disable the bandwidth server when you're done. It's a security risk if left open.

### IP scan and neighbors

```
/tool ip-scan interface=bridge1
/ip neighbor print              # MikroTik/CDP/LLDP neighbors
/ip arp print
```

## 4. Netwatch (up/down alerts and actions)

```
/tool netwatch add host=8.8.8.8 interval=10s timeout=1s type=icmp \
  down-script=":log warning \"INTERNET DOWN\"" \
  up-script=":log info \"INTERNET UP\""
/tool netwatch print
```

With email:

```
/tool e-mail set server=smtp.gmail.com port=587 tls=starttls user=you@gmail.com password="<app-password>" from=you@gmail.com
/tool netwatch set [find host=8.8.8.8] down-script="/tool e-mail send to=you@gmail.com subject=\"R1: Internet down\" body=\"8.8.8.8 unreachable\""
```

Use a Gmail **app password**, not your real password.

## 5. Graphs (built-in, viewed in WebFig)

```
/tool graphing interface add interface=ether1 allow-address=192.168.1.0/24
/tool graphing resource add allow-address=192.168.1.0/24
/tool graphing queue add allow-address=192.168.1.0/24
```

View them at `http://router-ip/graphs/`. The www service must be enabled and allowed from your network.

## 6. SNMP

```
/snmp community add name=labmon addresses=192.168.1.0/24 read-access=yes write-access=no
/snmp set enabled=yes contact="Fayez" location="EVE-NG lab" trap-version=2
```

Disable or restrict the default `public` community:

```
/snmp community set [find name=public] disabled=yes
```

SNMPv3 (secure):

```
/snmp community add name=v3user security=private authentication-protocol=SHA1 authentication-password=AuthPass123 encryption-protocol=AES encryption-password=EncPass123 addresses=192.168.1.0/24
```

Pollers: Zabbix, LibreNMS, PRTG, Grafana + Prometheus (SNMP exporter).

## 7. Traffic flow (NetFlow / IPFIX)

```
/ip traffic-flow set enabled=yes interfaces=ether1
/ip traffic-flow target add dst-address=192.168.1.108 port=2055 version=9
```

Collectors: ntopng, Elastiflow, PRTG.

## 8. The Dude

The Dude is MikroTik's free network monitoring and map tool.
- **Server:** the `dude` package on RouterOS (ARM, x86 and **CHR**). It needs disk space.
- **Client:** a Windows app, downloaded from mikrotik.com.

Install on a CHR:
1. Download `dude-7.x.x.npk` for the CHR (x86) architecture, the same version as RouterOS.
2. Upload it to the router (WinBox Files) and reboot.
3. Run:
   ```
   /dude set enabled=yes data-directory=dude
   ```
4. Open the Dude client and connect to the CHR IP. Run the **Discover** wizard to auto-map the network.

## EVE-NG lab

1. On R1: Netwatch on 8.8.8.8 with log up/down scripts. Disconnect the Cloud0 link in EVE-NG and watch the log.
2. Sniffer: capture a ping from PC1, download the pcap, and open it in Wireshark.
3. Remote syslog to your Windows PC (`192.168.1.108`).
4. Install The Dude on a separate CHR and discover your lab.
5. SNMP: walk R1 from your PC: `snmpwalk -v2c -c labmon 192.168.1.133` (comes with Net-SNMP).

## Troubleshooting

| Problem | Fix |
|---|---|
| Syslog server receives nothing | Windows firewall blocks UDP 514, or the router can't route to the PC. |
| SNMP timeout | Wrong `addresses=` on the community, or the firewall input chain drops UDP 161. |
| Graphs page 403 | `allow-address` doesn't include your PC, or www is disabled. |
| Netwatch flapping | Increase `timeout` and `interval`, or use `type=icmp` with `thr-loss-percent`. |
| Dude package not loading | Version or architecture mismatch with RouterOS. |
