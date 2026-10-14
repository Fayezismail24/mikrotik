# K. QoS: Queues and Mangle

> **Status:** 📝 Not yet tested in lab. Note the CHR free license limits upload to 1 Mbit/s per interface, so test with small limits like 256k and 512k.

## Concept

QoS controls **who gets how much bandwidth** and **what goes first** when a link is full.

| Tool | Use |
|---|---|
| **Simple queues** | Easy per-IP or per-subnet limits. Ordered list, first match wins. |
| **Queue tree** | Advanced hierarchical shaping using **packet marks** from mangle. |
| **PCQ** (queue type) | Splits a limit **fairly** among many users automatically. |
| **Mangle** | Marks connections and packets so queue trees know what is what. |

### Key terms

- **max-limit:** the hard ceiling (upload/download).
- **limit-at:** guaranteed (CIR) rate.
- **burst-limit / burst-threshold / burst-time:** a short speed boost above max-limit.
- **priority:** 1 (highest) to 8 (lowest). It only matters between siblings in the same parent.
- In simple queues, **target** = the client side. `max-limit=upload/download` is from the client's point of view.

> **Fasttrack bypasses queues and mangle.** Disable the fasttrack rule, or make it exclude the traffic you shape:
> ```
> /ip firewall filter disable [find action=fasttrack-connection]
> ```

---

## 1. Simple queues

Limit one PC to 2 Mbit/s up and 10 Mbit/s down:

```
/queue simple add name=PC1 target=192.168.10.10/32 max-limit=2M/10M
```

Limit a whole subnet:

```
/queue simple add name=LAN-total target=192.168.10.0/24 max-limit=20M/100M
```

Burst: 20M for the first few seconds, then down to 10M:

```
/queue simple add name=PC2 target=192.168.10.11/32 max-limit=2M/10M burst-limit=4M/20M burst-threshold=1500k/8M burst-time=16s/16s
```

Parent/child (users share the total):

```
/queue simple add name=TOTAL target=192.168.10.0/24 max-limit=20M/100M
/queue simple add name=PC1 target=192.168.10.10/32 parent=TOTAL max-limit=5M/30M
/queue simple add name=PC2 target=192.168.10.11/32 parent=TOTAL max-limit=5M/30M
```

Time-based limits (business hours):

```
/queue simple add name=Daytime target=192.168.10.0/24 max-limit=10M/50M time=8h-18h,mon,tue,wed,thu,fri
```

---

## 2. PCQ: fair sharing among all users

Each user gets an equal share of 100M, adjusting dynamically as users join:

```
/queue type add name=pcq-down kind=pcq pcq-classifier=dst-address pcq-rate=0
/queue type add name=pcq-up   kind=pcq pcq-classifier=src-address pcq-rate=0
/queue simple add name=LAN-fair target=192.168.10.0/24 max-limit=20M/100M queue=pcq-up/pcq-down
```

Fixed cap per user (each user gets at most 5M down):

```
/queue type set pcq-down pcq-rate=5M
```

---

## 3. Queue tree with mangle (prioritize VoIP and gaming)

### Mangle: classify traffic

Mark connections first (cheap), then packets from the connection mark:

```
/ip firewall mangle
add chain=forward protocol=udp port=5060,10000-20000 connection-state=new action=mark-connection new-connection-mark=VOIP passthrough=yes comment="VoIP"
add chain=forward connection-mark=VOIP action=mark-packet new-packet-mark=VOIP-P passthrough=no

add chain=forward protocol=tcp dst-port=80,443 connection-state=new action=mark-connection new-connection-mark=WEB passthrough=yes comment="Web"
add chain=forward connection-mark=WEB action=mark-packet new-packet-mark=WEB-P passthrough=no

add chain=forward connection-mark=no-mark action=mark-connection new-connection-mark=OTHER passthrough=yes comment="Everything else"
add chain=forward connection-mark=OTHER action=mark-packet new-packet-mark=OTHER-P passthrough=no
```

### Queue tree: download (out the LAN interface)

```
/queue tree
add name=DOWN parent=bridge1 max-limit=100M
add name=DOWN-VOIP  parent=DOWN packet-mark=VOIP-P  priority=1 limit-at=2M  max-limit=100M
add name=DOWN-WEB   parent=DOWN packet-mark=WEB-P   priority=4 limit-at=20M max-limit=100M queue=pcq-down
add name=DOWN-OTHER parent=DOWN packet-mark=OTHER-P priority=8 limit-at=5M  max-limit=100M queue=pcq-down
```

### Queue tree: upload (out the WAN interface)

```
/queue tree
add name=UP parent=ether1 max-limit=20M
add name=UP-VOIP  parent=UP packet-mark=VOIP-P  priority=1 limit-at=1M max-limit=20M
add name=UP-WEB   parent=UP packet-mark=WEB-P   priority=4 limit-at=5M max-limit=20M queue=pcq-up
add name=UP-OTHER parent=UP packet-mark=OTHER-P priority=8 limit-at=1M max-limit=20M queue=pcq-up
```

**Tip:** set `max-limit` about 90% of your real line speed, so the queue happens in **your** router and not in the ISP modem, where you have no control.

`parent=global` is another option: a tree that sees traffic in all directions, but you then need separate marks for up and down.

---

## 4. CAKE / fq_codel (bufferbloat)

RouterOS 7 has modern AQM queue types. A simple, very effective setup:

```
/queue type add name=cake-down kind=cake cake-bandwidth=90M cake-diffserv=diffserv4 cake-nat=yes
/queue type add name=cake-up   kind=cake cake-bandwidth=18M cake-diffserv=diffserv4 cake-nat=yes
/queue simple add name=WAN-cake target=ether1 queue=cake-up/cake-down max-limit=18M/90M
```

`target=ether1` here means "all traffic through the WAN interface". Test bufferbloat on waveform.com/tools/bufferbloat.

---

## EVE-NG lab

```
[Cloud0] --ether1-- R1 --ether2 (bridge1)-- CHR-client1 (192.168.10.10)
                                          \- CHR-client2 (192.168.10.11)
```

Use CHR nodes as clients, since VPCS can't generate bulk traffic. The RouterOS **bandwidth test** works between CHRs:

1. On R1: `/tool bandwidth-server set enabled=yes`.
2. Disable fasttrack.
3. Add simple queue `PC1` = `256k/512k`.
4. On client1: `/tool bandwidth-test address=192.168.10.1 direction=receive protocol=tcp duration=20s`. You should see about 512k.
5. Watch live: `/queue simple print stats` and WinBox Queues (colored bars).
6. Replace it with PCQ, run the test on both clients at once, and confirm the bandwidth splits about evenly.

## Verify

```
/queue simple print stats
/queue tree print stats
/ip firewall mangle print stats
/tool torch interface=bridge1
```

## Troubleshooting

| Problem | Fix |
|---|---|
| Queue never fills, no limiting | Fasttrack is active, or the target doesn't match (NATed address, wrong subnet). |
| Up and down limits swapped | Simple queue `max-limit` = **client upload/download**. |
| Queue tree has 0 traffic | Packet marks don't match: check mangle stats, or the wrong parent interface. |
| Everything is slow after QoS | max-limit set higher than the real line rate, or limit-at sums exceed the parent. |
| CHR test stuck around 1M | The CHR free license limits upload to 1 Mbit/s per interface. |
