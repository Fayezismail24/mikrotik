# K. QoS: Queues and Mangle

> **Status:** 📝 Not yet tested. The CHR free license limits upload to 1 Mbit/s per interface, so test with small limits like 256k and 512k.
>
> **Cert:** MTCNA module 7 (QoS) · MTCTCE · 🔹 CAKE is beyond MTCNA

## Concept

| Tool | Use |
|---|---|
| **Simple queues** | Easy per-IP or per-subnet limits. First match wins. |
| **Queue tree** | Hierarchical shaping using **packet marks** from mangle. |
| **PCQ** | Splits a limit **fairly** among many users. |
| **Mangle** | Marks connections and packets for queue trees. |

Key terms:
- **max-limit:** the ceiling.
- **limit-at:** the guaranteed rate.
- **burst:** a short boost above max-limit.
- **priority:** 1 (highest) to 8.
- In simple queues, `max-limit=upload/download` is from the **client's** point of view.

🖱️ **In WinBox:** **Queues** has the tabs Simple Queues, Interface Queues, Queue Tree, Queue Types. Mangle is at **IP → Firewall → Mangle**.

> **Fasttrack bypasses queues.** Disable it first:
>
> 🖱️ IP → Firewall → Filter Rules → select the fasttrack rule → **✕**
>
> ⌨️ `/ip firewall filter disable [find action=fasttrack-connection]`

---

## 1. Simple queues

### One PC: 2M up / 10M down

🖱️ **WinBox:** Queues → **Simple Queues** → **+**:
- **General** tab:
  - Name `PC1`
  - Target `192.168.10.10/32`
  - Max Limit: Upload `2M`, Download `10M`

Click **OK**.

⌨️ **CLI:**
```
/queue simple add name=PC1 target=192.168.10.10/32 max-limit=2M/10M
```

### Whole subnet

🖱️ **WinBox:** same as above, with Target `192.168.10.0/24` and Max Limit `20M` / `100M`.

⌨️ **CLI:**
```
/queue simple add name=LAN-total target=192.168.10.0/24 max-limit=20M/100M
```

### Burst

🖱️ **WinBox:** Simple Queues → **+** → General tab:

| Field | Upload | Download |
|---|---|---|
| Max Limit | `2M` | `10M` |
| Burst Limit | `4M` | `20M` |
| Burst Threshold | `1500k` | `8M` |
| Burst Time | `16` s | `16` s |

⌨️ **CLI:**
```
/queue simple add name=PC2 target=192.168.10.11/32 max-limit=2M/10M burst-limit=4M/20M burst-threshold=1500k/8M burst-time=16s/16s
```

### Parent / child

🖱️ **WinBox:**
1. **+** → Name `TOTAL`, Target `192.168.10.0/24`, Max Limit `20M` / `100M`.
2. **+** → Name `PC1`, Target `.10/32`, Max Limit `5M` / `30M`. On the **Advanced** tab: Parent `TOTAL`.

⌨️ **CLI:**
```
/queue simple add name=TOTAL target=192.168.10.0/24 max-limit=20M/100M
/queue simple add name=PC1 target=192.168.10.10/32 parent=TOTAL max-limit=5M/30M
/queue simple add name=PC2 target=192.168.10.11/32 parent=TOTAL max-limit=5M/30M
```

### Time-based

🖱️ **WinBox:** Simple Queues → **+** → **Advanced** tab → Time `08:00:00`–`18:00:00`, tick Mon–Fri.

⌨️ **CLI:**
```
/queue simple add name=Daytime target=192.168.10.0/24 max-limit=10M/50M time=8h-18h,mon,tue,wed,thu,fri
```

---

## 2. PCQ: fair sharing

### Step 1: queue types

🖱️ **WinBox:** Queues → **Queue Types** → **+**:
- Type Name `pcq-down`, Kind `pcq`, Rate `0`, Classifier: **Dst. Address** ✓.
- Type Name `pcq-up`, Kind `pcq`, Rate `0`, Classifier: **Src. Address** ✓.

⌨️ **CLI:**
```
/queue type add name=pcq-down kind=pcq pcq-classifier=dst-address pcq-rate=0
/queue type add name=pcq-up   kind=pcq pcq-classifier=src-address pcq-rate=0
```

### Step 2: use them

🖱️ **WinBox:** Simple Queues → **+**:
- General: Target `192.168.10.0/24`, Max Limit `20M` / `100M`.
- **Queues** tab: Upload Queue Type `pcq-up`, Download Queue Type `pcq-down`.

⌨️ **CLI:**
```
/queue simple add name=LAN-fair target=192.168.10.0/24 max-limit=20M/100M queue=pcq-up/pcq-down
```

To cap each user at 5M: set Rate to `5M` on `pcq-down` (`/queue type set pcq-down pcq-rate=5M`).

---

## 3. Queue tree with mangle (VoIP first)

### Step 1: mangle marks

For each class, add **two** rules: first mark the connection, then mark its packets.

🖱️ **WinBox:** IP → Firewall → **Mangle** → **+**:

| # | General | Action tab |
|---|---|---|
| 1 | Chain `forward`, Protocol `udp`, Port `5060,10000-20000`, Connection State `new` | mark connection → `VOIP`, Passthrough ✓ |
| 2 | Chain `forward`, Connection Mark `VOIP` | mark packet → `VOIP-P`, Passthrough ✗ |
| 3 | Chain `forward`, Protocol `tcp`, Dst. Port `80,443`, Connection State `new` | mark connection → `WEB`, Passthrough ✓ |
| 4 | Chain `forward`, Connection Mark `WEB` | mark packet → `WEB-P`, Passthrough ✗ |
| 5 | Chain `forward`, Connection Mark `no-mark` | mark connection → `OTHER`, Passthrough ✓ |
| 6 | Chain `forward`, Connection Mark `OTHER` | mark packet → `OTHER-P`, Passthrough ✗ |

⌨️ **CLI:**
```
/ip firewall mangle
add chain=forward protocol=udp port=5060,10000-20000 connection-state=new action=mark-connection new-connection-mark=VOIP passthrough=yes comment="VoIP"
add chain=forward connection-mark=VOIP action=mark-packet new-packet-mark=VOIP-P passthrough=no
add chain=forward protocol=tcp dst-port=80,443 connection-state=new action=mark-connection new-connection-mark=WEB passthrough=yes comment="Web"
add chain=forward connection-mark=WEB action=mark-packet new-packet-mark=WEB-P passthrough=no
add chain=forward connection-mark=no-mark action=mark-connection new-connection-mark=OTHER passthrough=yes comment="Other"
add chain=forward connection-mark=OTHER action=mark-packet new-packet-mark=OTHER-P passthrough=no
```

### Step 2: queue tree (download, out the LAN)

🖱️ **WinBox:** Queues → **Queue Tree** → **+**:

| Name | Parent | Packet Marks | Priority | Limit At | Max Limit | Queue Type |
|---|---|---|---|---|---|---|
| DOWN | `bridge1` | | | | 100M | |
| DOWN-VOIP | DOWN | VOIP-P | 1 | 2M | 100M | default |
| DOWN-WEB | DOWN | WEB-P | 4 | 20M | 100M | pcq-down |
| DOWN-OTHER | DOWN | OTHER-P | 8 | 5M | 100M | pcq-down |

⌨️ **CLI:**
```
/queue tree
add name=DOWN parent=bridge1 max-limit=100M
add name=DOWN-VOIP  parent=DOWN packet-mark=VOIP-P  priority=1 limit-at=2M  max-limit=100M
add name=DOWN-WEB   parent=DOWN packet-mark=WEB-P   priority=4 limit-at=20M max-limit=100M queue=pcq-down
add name=DOWN-OTHER parent=DOWN packet-mark=OTHER-P priority=8 limit-at=5M  max-limit=100M queue=pcq-down
```

### Step 3: queue tree (upload, out the WAN)

🖱️ **WinBox:** same as step 2 with Parent `ether1`: `UP` (20M), `UP-VOIP` (prio 1), `UP-WEB` (prio 4, `pcq-up`), `UP-OTHER` (prio 8, `pcq-up`).

⌨️ **CLI:**
```
/queue tree
add name=UP parent=ether1 max-limit=20M
add name=UP-VOIP  parent=UP packet-mark=VOIP-P  priority=1 limit-at=1M max-limit=20M
add name=UP-WEB   parent=UP packet-mark=WEB-P   priority=4 limit-at=5M max-limit=20M queue=pcq-up
add name=UP-OTHER parent=UP packet-mark=OTHER-P priority=8 limit-at=1M max-limit=20M queue=pcq-up
```

**Tip:** set max-limit to about 90% of your real line speed, so the queueing happens in **your** router.

---

## 4. CAKE (fixes bufferbloat)

### Step 1: queue types

🖱️ **WinBox:** Queues → Queue Types → **+**:
- `cake-down`: Kind `cake`, Bandwidth `90M`, Diffserv `diffserv4`, NAT ✓.
- `cake-up`: Kind `cake`, Bandwidth `18M`, Diffserv `diffserv4`, NAT ✓.

⌨️ **CLI:**
```
/queue type add name=cake-down kind=cake cake-bandwidth=90M cake-diffserv=diffserv4 cake-nat=yes
/queue type add name=cake-up   kind=cake cake-bandwidth=18M cake-diffserv=diffserv4 cake-nat=yes
```

### Step 2: apply to the WAN

🖱️ **WinBox:** Simple Queues → **+**:
- Target `ether1`, Max Limit `18M` / `90M`.
- Queues tab: `cake-up` / `cake-down`.

⌨️ **CLI:**
```
/queue simple add name=WAN-cake target=ether1 queue=cake-up/cake-down max-limit=18M/90M
```

Test the result at waveform.com/tools/bufferbloat.

---

## EVE-NG lab

```
[Cloud0] --ether1-- R1 --ether2 (bridge1)-- CHR-client1 (192.168.10.10)
                                          \- CHR-client2 (192.168.10.11)
```

1. On R1, enable the bandwidth test server. 🖱️ Tools → **BTest Server** → Enabled ✓. ⌨️ `/tool bandwidth-server set enabled=yes`
2. Disable fasttrack.
3. Add a simple queue `PC1` = `256k/512k`.
4. Test from client1. 🖱️ Tools → **Bandwidth Test** → Address `192.168.10.1`, Direction `receive`, Protocol `tcp` → Start. ⌨️ `/tool bandwidth-test address=192.168.10.1 direction=receive protocol=tcp duration=20s`. Expect about 512k.
5. Watch the queue bars live in Queues → Simple Queues.
6. Switch to PCQ, test both clients at once, and confirm the split is about equal.

## Verify

| Check | 🖱️ WinBox | ⌨️ CLI |
|---|---|---|
| Simple queue usage | Queues → Simple Queues (colored bars, Rate column) | `/queue simple print stats` |
| Queue tree | Queues → Queue Tree | `/queue tree print stats` |
| Mangle counters | IP → Firewall → Mangle | `/ip firewall mangle print stats` |
| Live traffic | Tools → Torch | `/tool torch interface=bridge1` |

## Troubleshooting

| Problem | Fix |
|---|---|
| No limiting | Fasttrack is on, or the target doesn't match. |
| Up and down swapped | Simple queue = **client** upload/download. |
| Queue tree shows 0 | Packet marks don't match, or the wrong parent. |
| Everything slow | max-limit above the real line rate, or limit-at sums exceed the parent. |
| Stuck around 1M on CHR | Free license limit. |
