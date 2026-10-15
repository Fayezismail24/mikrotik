# Lab 1 Q&A: DHCP Server, Bridge and NAT

> Follow-up "what if / why" questions from reviewing [Lab 1](README.md). Try each one before opening the answer.

[Lab 1](README.md) · [Labs index](../README.md) · [Back to the main index](../../README.md)

## Contents

| Section | Questions |
|---|---|
| [WAN side: Cloud0 and the DHCP client](#wan-side-cloud0-and-the-dhcp-client) | 1–4 |
| [LAN side: the bridge](#lan-side-the-bridge) | 5–8 |
| [LAN side: the IP on the bridge](#lan-side-the-ip-on-the-bridge) | 9–11 |
| [DHCP: pool and network entry](#dhcp-pool-and-network-entry) | 12–16 |
| [DHCP: how it works](#dhcp-how-it-works) | 17–22 |
| [NAT](#nat) | 23–30 |
| [Routing and DNS](#routing-and-dns) | 31–32 |
| [Bridge extras](#bridge-extras) | 33–35 |
| [Security basics](#security-basics) | 36–37 |
| [Troubleshooting scenarios](#troubleshooting-scenarios) | 38–41 |
| [EVE-NG specifics](#eve-ng-specifics) | 42–44 |
| [MTCNA exam-style](#mtcna-exam-style) | 45–52 |

---

## WAN side: Cloud0 and the DHCP client

**1. What is Cloud0 really? Is it a switch?**

<details><summary>Answer</summary>

Cloud0 is EVE-NG's link to `pnet0`, a Linux bridge on the EVE-NG server that is attached to the server's real network card. A bridge behaves like a switch, so cabling R1 ether1 to Cloud0 is like plugging R1 into a spare port on the home router. It is not a separate switch device; it is a doorway from the lab to the real home LAN.

</details>

**2. R1 is a DHCP client and a DHCP server at the same time. How?**

<details><summary>Answer</summary>

Different interfaces, different roles.

| | DHCP client | DHCP server |
|---|---|---|
| Role | **Asks** for an IP | **Gives** IPs |
| On R1 | ether1 (WAN) | bridge1 (LAN) |
| Talks to | Home router answers R1 | VPCs ask R1 |

From the home router R1 receives three things: an IP (`192.168.1.133/24`), a default gateway (`192.168.1.1`, the `0.0.0.0/0` route with flags `DAd`) and DNS (`192.168.1.1`).

</details>

**3. The Cloud0 link goes down. What does R1 lose, and what still works for the VPCs?**

<details><summary>Answer</summary>

The DHCP client on ether1 is no longer bound, so R1 loses all three things it got from the home router:

| Lost | Effect |
|---|---|
| WAN IP | R1 has no address on the home LAN |
| Default route | Nowhere to send internet traffic. This is why `8.8.8.8` fails |
| DNS | Names don't resolve |

The NAT rule is still there but useless: masquerade needs a route out.

Ping ladder from a VPC (test from the inside out; the first ❌ shows where it broke):

```
ping 192.168.2.x  (other VPC) ✅  bridge (Layer 2)
ping 192.168.2.1  (R1)        ✅  R1's bridge IP
ping 192.168.1.1  (home rtr)  ❌  no WAN IP / route
ping 8.8.8.8                  ❌  no default route
ping google.com               ❌  no DNS, no route
```

</details>

**4. Why must you never run a DHCP server on the port connected to Cloud0?**

<details><summary>Answer</summary>

That port is on your real home LAN. Your phones, laptops and TV ask for an IP by broadcast, and both the home router and R1 would answer. Whoever answers first wins, so some devices would get a `192.168.2.x` address with R1 as gateway, or an address that clashes with the real network. That is a **rogue DHCP server**, and it breaks the home network for everyone.

</details>

## LAN side: the bridge

**5. Why does the DHCP server go on a bridge instead of on each port?**

<details><summary>Answer</summary>

DHCP requests are broadcasts, and a broadcast stays inside one Layer-2 network. The bridge joins ether2–ether5 into **one** network, so one IP, one pool and one DHCP server cover all four VPCs. Without the bridge, each port is its own network and needs its own subnet, IP, pool and server.

| | Separate ports | Bridge |
|---|---|---|
| Networks | One per port | One shared |
| Router IPs | One per port, different subnets | One, on the bridge |
| DHCP servers | One per port | One |
| PC to PC traffic | Routed by R1 (Layer 3) | Switched by the bridge (Layer 2) |

</details>

**6. VPC1 pings VPC2. Is that traffic routed by R1 or switched by the bridge? How can you prove it?**

<details><summary>Answer</summary>

**Switched.** Both are in `192.168.2.0/24`, so VPC1 doesn't use its gateway at all; it ARPs for VPC2 directly.

Interface traffic counters on ether2/ether3 do **not** prove it: they rise whether traffic is switched or routed. These do:

| Check | Command | Proof |
|---|---|---|
| ARP on the VPC | VPC1: `show arp` | VPC2's IP maps to **VPC2's MAC**, not R1's |
| Traceroute | VPC1: `trace 192.168.2.x` | One hop, no `192.168.2.1` in between |
| Bridge MAC table | 🖱️ Bridge → **Hosts** · ⌨️ `/interface bridge host print` | VPC MACs learned on their ports: the switch's forwarding table |

Rule: **same subnet = switched (L2, MAC); different subnet = routed (L3, gateway).**

</details>

**7. Where do you see R1's MAC address, and which one do the VPCs use as gateway?**

<details><summary>Answer</summary>

🖱️ Interfaces → double-click the interface → **General** → MAC Address. Or right-click a column header → **Show Columns** → MAC Address to see all of them.
⌨️ `/interface print detail` · `/interface bridge print`

The VPCs use **bridge1's MAC**, because bridge1 owns `192.168.2.1`. With `auto-mac=yes` the bridge borrows a member port's MAC (usually the first added), so bridge1 and ether2 often show the same MAC. That is normal.

Test: on VPC1, `ping 8.8.8.8` then `show arp` shows bridge1's MAC for `192.168.2.1` (routed). `ping` VPC2 then `show arp` shows VPC2's own MAC (switched).

</details>

**8. ether2 joins the bridge but the DHCP server is still on ether2. What happens?**

<details><summary>Answer</summary>

ether2 becomes a slave port (`S` flag). The server turns **invalid** (`I` flag, red in WinBox) with `;;; DHCP server can not run on slave interface!`, and VPCs get no IP.

Why: the bridge owns everything arriving on its member ports. The DHCP Discover lands on bridge1, so a server listening on ether2 never sees it.

Fix:
```
/ip dhcp-server set server1 interface=bridge1
/ip address set [find interface=ether2] interface=bridge1
```

Watch out: an **IP address** left on a slave port gives **no** warning. It just doesn't work properly.

</details>

## LAN side: the IP on the bridge

**9. What does the IP `192.168.2.1/24` on bridge1 give you?**

<details><summary>Answer</summary>

| The IP gives | Without it |
|---|---|
| Gateway `.2.1` for the VPCs | VPCs have no way out of their subnet |
| Connected route `192.168.2.0/24` via bridge1 | R1 forgets the LAN exists, so replies from the internet have nowhere to go, even for static-IP PCs |
| A source address for DHCP replies | The DHCP server can't hand out leases |

It must be a **different subnet** from ether1 (`192.168.1.0/24`), or R1 can't tell which side a host is on.

Try it safely: IP → Addresses → **disable** `192.168.2.1/24`, run `clear ip` + `ip dhcp` on a VPC, check `/log print where topics~"dhcp"` and IP → Routes, then re-enable.

</details>

**10. How do you read the DHCP log?**

<details><summary>Answer</summary>

```
dhcp,info client1 on ether1 got IP address 192.168.1.133
dhcp,info server1 deassigned 192.168.2.8 for 00:50:79:66:68:04 VPCS1
dhcp,info server1 assigned 192.168.2.8 for 00:50:79:66:68:04 VPCS1
```

- `client1 ... got IP address`: R1's DHCP **client** on the WAN got its lease.
- `deassigned` then `assigned` a second later: a VPC ran `clear ip` then `ip dhcp`.
- The same IP comes back because the server tracks clients by **MAC address**.
- `VPCS1` is the hostname the client sends. Every VPCS node sends the same default name, so identify clients by MAC (`00:50:79:66:68:xx` is the VPCS prefix) or in IP → DHCP Server → **Leases**.

</details>

**11. What do the route flags mean?**

<details><summary>Answer</summary>

| Flag | Meaning |
|---|---|
| **D** dynamic | Created automatically, not typed in; can't be edited or deleted directly |
| **A** active | Currently used for forwarding (an inactive route is listed but not used, e.g. interface down) |
| **C** connected | Network directly attached, from an IP on one of R1's interfaces |
| **d** | Learned from DHCP |
| **s** | Static, added by hand |

| Route in this lab | Flags | Source |
|---|---|---|
| `192.168.2.0/24` via bridge1 | DAC | IP on bridge1 |
| `192.168.1.0/24` via ether1 | DAC | IP from DHCP on ether1 |
| `0.0.0.0/0` via 192.168.1.1 | DAd | Default gateway handed out by the home router |

</details>

## DHCP: pool and network entry

**12. The pool is `192.168.2.2-192.168.2.10`. How many clients can get an IP, and what happens to the 10th?**

<details><summary>Answer</summary>

`.2` to `.10` is **9 addresses**, so 9 clients at once. The 10th gets **nothing**: its `ip dhcp` times out, and R1 logs that the pool is empty (`/log print where topics~"dhcp"`).

An address only goes back to the pool when its lease **expires** or is released. Fixes: a bigger pool (`ranges=192.168.2.2-192.168.2.254`), a shorter lease time, or a second pool chained with `next-pool`.

</details>

**13. Pool vs. network entry: what is the difference?**

<details><summary>Answer</summary>

| | Pool | Network entry |
|---|---|---|
| Answers | **Which IP** does the client get? | **What else** does it need? |
| Contains | A range: `192.168.2.2-192.168.2.10` | Subnet, gateway, DNS: `192.168.2.0/24`, `.2.1`, `8.8.8.8` |
| Linked by | `address-pool=pool1` on the server | Matched automatically by subnet |
| Missing → | Server only answers static leases (`static-only`) | Client gets an IP with `/0` mask and gateway `0.0.0.0` |

Analogy: the pool is the stack of hotel room keys; the network entry is the info card handed over with each key.

🖱️ IP → Pool · IP → DHCP Server → **Networks**
⌨️ `/ip pool print` · `/ip dhcp-server network print`

</details>

**14. You remove only the gateway from the network entry and a VPC renews. Which pings still work?**

<details><summary>Answer</summary>

The VPC gets its IP and `/24` mask, but gateway `0.0.0.0`. It reaches anything **in its own subnet** and nothing outside it.

```
ping 192.168.2.x  (other VPC) ✅  same subnet, no gateway needed
ping 192.168.2.1  (R1)        ✅  same subnet
ping 192.168.1.1              ❌  different subnet, nowhere to send it
ping 8.8.8.8                  ❌
```

R1 is fine; the PC just doesn't know where the exit is.

</details>

**15. What happens if you leave DNS empty in the network entry?**

<details><summary>Answer</summary>

R1 hands out **its own** DNS servers instead: the ones in IP → DNS, which here came from the home router (`192.168.1.1`) through the DHCP client on ether1. Names still resolve.

Set it explicitly (`dns-server=8.8.8.8`) when you want clients to use a specific resolver.

</details>

**16. Why does the pool start at `.2` and not `.1`?**

<details><summary>Answer</summary>

`.1` is R1's own address on bridge1. If the pool included it, the server could hand the router's address to a PC: a **duplicate IP** that breaks the gateway for everyone. Keep the router's IP, and any other fixed IPs (printers, servers), **outside** the pool.

</details>

## DHCP: how it works

**17. What are the four DHCP messages, and why are some broadcast?**

<details><summary>Answer</summary>

**DORA:**

| Step | From → To | Why |
|---|---|---|
| **D**iscover | Client → broadcast `255.255.255.255` | Client has no IP and doesn't know where the server is |
| **O**ffer | Server → client | "You can have `192.168.2.8`" |
| **R**equest | Client → broadcast | "I'll take `.8` from this server" (also tells other servers they weren't chosen) |
| **A**ck | Server → client | "Confirmed: here's your lease, gateway and DNS" |

Ports: client **UDP 68**, server **UDP 67**. Because Discover is a broadcast, it never crosses a router: the DHCP server must be on the same Layer-2 network as its clients, or a DHCP relay is needed.

</details>

**18. What is the lease time, and what happens when it runs out?**

<details><summary>Answer</summary>

How long a client may keep its IP (RouterOS v7 default: **30 minutes**). The client tries to **renew** at half the lease time, directly with the server; if R1 answers, the lease is extended and the IP stays the same. If the lease fully expires, the address goes back into the pool.

🖱️ IP → DHCP Server → server1 → **Lease Time**
⌨️ `/ip dhcp-server set server1 lease-time=1h`

Short leases free addresses quickly (busy guest Wi-Fi); long leases mean fewer renewals (stable office).

</details>

**19. Where do you see which PC got which IP?**

<details><summary>Answer</summary>

🖱️ IP → DHCP Server → **Leases**
⌨️ `/ip dhcp-server lease print`

Each row shows address, MAC, hostname and status. **D** = dynamic (from the pool). Status **bound** = in use; **waiting** = a static lease whose client hasn't asked yet.

</details>

**20. How do you make a PC always get the same IP?**

<details><summary>Answer</summary>

Turn its lease into a **static lease**, a reservation tied to its MAC:

🖱️ IP → DHCP Server → **Leases** → select the lease → **Make Static**. Double-click it to change the address if you want.
⌨️
```
/ip dhcp-server lease make-static [find address=192.168.2.8]
/ip dhcp-server lease set [find mac-address=00:50:79:66:68:04] address=192.168.2.50
```

The `D` flag disappears. A static lease can be **outside** the pool (e.g. `.50`), which keeps reserved addresses apart from the dynamic range.

</details>

**21. What does `address-pool=static-only` mean?**

<details><summary>Answer</summary>

The server answers **only** clients that have a static lease; everyone else gets nothing. Useful when only known devices should get an IP. It's also why your VPCs got nothing before `pool1` was linked.

</details>

**22. What happens if there are two DHCP servers on the same LAN?**

<details><summary>Answer</summary>

Both answer every Discover and the client takes the **first Offer** to arrive. Some clients end up with the wrong subnet or gateway, and the problem looks random. An unwanted second server is a **rogue DHCP server** (e.g. a home router plugged in backwards, or R1's server on the Cloud0 port).

RouterOS can warn you: `/ip dhcp-server alert add interface=bridge1 valid-server=<MAC of the real server>` logs any other server that answers.

</details>

## NAT

**23. What exactly does masquerade change in the packet?**

<details><summary>Answer</summary>

On the way **out** it replaces the **source IP** (`192.168.2.8`) with ether1's IP (`192.168.1.133`), and changes the source port if needed. Connection tracking remembers the swap. On the way **back** the reply is addressed to `192.168.1.133`; R1 looks it up and rewrites the destination back to `192.168.2.8`.

```
Out:  192.168.2.8:50000 → 8.8.8.8    becomes  192.168.1.133:50000 → 8.8.8.8
Back: 8.8.8.8 → 192.168.1.133:50000  becomes  8.8.8.8 → 192.168.2.8:50000
```

</details>

**24. srcnat vs. dstnat?**

<details><summary>Answer</summary>

| | srcnat | dstnat |
|---|---|---|
| Changes | **Source** address | **Destination** address |
| When | Packet **leaving** the router | Packet **arriving** at the router |
| Typical use | LAN to internet (masquerade) | Port forwarding (internet to a LAN server) |
| This lab | ✅ masquerade out ether1 | Not used yet (section F) |

</details>

**25. masquerade vs. `action=src-nat to-addresses=...`?**

<details><summary>Answer</summary>

Both do source NAT.

| | masquerade | src-nat |
|---|---|---|
| Source IP used | Whatever IP the out-interface has **right now** | A fixed IP you type |
| Best for | **Dynamic** WAN (DHCP, PPPoE), like our ether1 | **Static** public IP |
| If the WAN IP changes | Follows it automatically and clears old connections | Breaks until you edit the rule |

ether1 gets its IP by DHCP, so masquerade is the right choice here.

</details>

**26. Why set `out-interface=ether1` on the NAT rule?**

<details><summary>Answer</summary>

So NAT only touches traffic **going to the WAN**. Without it, every routed packet leaving R1 through any interface is masqueraded. Harmless with one LAN, but once you add a second LAN or a VPN, traffic between internal networks would be NATed too and you'd lose the real source IPs (bad for logs and firewall rules).

</details>

**27. When VPC1 pings VPC2, does the packet go through NAT?**

<details><summary>Answer</summary>

No. VPC1 to VPC2 is switched inside bridge1 at Layer 2, so it never reaches R1's routing or the IP firewall (unless the bridge has `use-ip-firewall=yes`). And routed traffic is only masqueraded when it leaves via ether1. The NAT counter doesn't move during that ping.

</details>

**28. Is this double NAT? Does it matter?**

<details><summary>Answer</summary>

Yes. A VPC's packet is NATed twice:

```
192.168.2.8 → (R1 masquerade) → 192.168.1.133 → (home router NAT) → your public IP
```

Browsing and pings work fine. It matters for **incoming** connections: a port forward would be needed on **both** routers, and some apps (certain games, VoIP) can struggle. In a lab it's normal.

</details>

**29. Can a laptop on the home LAN ping VPC1 (`192.168.2.8`)? Why not?**

<details><summary>Answer</summary>

No. The laptop sends to its gateway (the home router), which has **no route** to `192.168.2.0/24`, so it sends the packet out to the ISP, where it's dropped. NAT lets the VPCs start connections **out**; it doesn't open a way **in**.

Fix, if you want it: on the home router add a static route `192.168.2.0/24 via 192.168.1.133`. R1 then routes the traffic in.

</details>

**30. How do you see NAT working?**

<details><summary>Answer</summary>

| What | 🖱️ WinBox | ⌨️ CLI | What you see |
|---|---|---|---|
| Rule counters | IP → Firewall → NAT (Bytes / Packets columns) | `/ip firewall nat print stats` | Counters rise while a VPC pings `8.8.8.8` |
| Translated connections | IP → Firewall → **Connections** | `/ip firewall connection print where src-address~"192.168.2."` | Source `192.168.2.x`, reply to `192.168.1.133`, flag **N** (srcnat) |

Tip: `/ip firewall nat reset-counters-all` before a test makes the change obvious.

</details>

## Routing and DNS

**31. Where does R1's default route come from, and what if it's missing?**

<details><summary>Answer</summary>

The DHCP client on ether1 has `add-default-route=yes` (the default), so it installs `0.0.0.0/0 via 192.168.1.1` with flags **DAd** (d = from DHCP).

If it's missing (e.g. `add-default-route=no`): R1 and the VPCs still reach `192.168.1.x` (connected route), but `8.8.8.8` fails because R1 has no route to it.

🖱️ IP → DHCP Client → ether1 → **Add Default Route** = yes
⌨️ `/ip dhcp-client set [find interface=ether1] add-default-route=yes`, or a static route: `/ip route add dst-address=0.0.0.0/0 gateway=192.168.1.1`

</details>

**32. Could the VPCs use R1 as their DNS server instead of `8.8.8.8`?**

<details><summary>Answer</summary>

Yes:
```
/ip dns set allow-remote-requests=yes
/ip dhcp-server network set [find address=192.168.2.0/24] dns-server=192.168.2.1
```
R1 then answers DNS for the LAN and caches results, so repeat lookups are faster.

⚠️ `allow-remote-requests=yes` also answers DNS arriving on ether1 unless a firewall blocks it. Drop UDP/TCP 53 coming in on the WAN in the **input** chain, or R1 becomes an open resolver.

</details>

## Bridge extras

**33. Does the bridge run spanning tree? Why would it need to?**

<details><summary>Answer</summary>

Yes. A RouterOS bridge uses **RSTP** by default (`protocol-mode=rstp`). If two of its ports were cabled into the same switch, broadcasts would loop forever (a broadcast storm). RSTP detects the loop and blocks one port. With four VPCs cabled straight to R1 there's no loop, but it's on by default for safety.

⌨️ `/interface bridge print` (see `protocol-mode`) · `/interface bridge port print` (port role and state)

</details>

**34. `auto-mac` and `admin-mac`: why set a fixed bridge MAC?**

<details><summary>Answer</summary>

With `auto-mac=yes` the bridge borrows a member port's MAC. If you later remove or reorder ports, the bridge MAC can **change**, and clients' ARP entries for `192.168.2.1` go stale until they refresh. A fixed MAC avoids that:

```
/interface bridge set bridge1 auto-mac=no admin-mac=<ether2's MAC>
```

</details>

**35. What happens if you accidentally add ether1 to bridge1?**

<details><summary>Answer</summary>

WAN and LAN become **one Layer-2 network**:
- ether1 becomes a slave port, so its DHCP client stops working: R1 loses its WAN IP and default route.
- bridge1's DHCP server now hears the **home LAN** too and starts answering your phones and laptops: a rogue DHCP server on your real network.
- You may lose WinBox access via `192.168.1.133`.

Fix: remove ether1 from the bridge (Bridge → Ports, or `/interface bridge port remove [find interface=ether1]`). If you're locked out, use the EVE-NG console, or WinBox **by MAC** from the Neighbors tab.

</details>

## Security basics

**36. CHR starts with no firewall. What on R1 is exposed to the home LAN?**

<details><summary>Answer</summary>

Every service listening on `192.168.1.133`: WinBox (8291), SSH (22), WebFig (80), Telnet (23), FTP (21), API (8728), and DNS (53) if remote requests are on. Anyone on the home LAN can try to log in.

Minimum hardening (section O):
- A strong admin password (CHR v7 asks for one on first login).
- Disable unused services: IP → **Services** · `/ip service disable telnet,ftp,www,api,api-ssl`
- Limit who can manage it: `/ip service set winbox address=192.168.1.0/24`
- Input-chain firewall rules (next question).

</details>

**37. input vs. forward chain: which one protects R1 itself?**

<details><summary>Answer</summary>

| Chain | Traffic | Example |
|---|---|---|
| **input** | **To** the router itself | WinBox to `192.168.1.133`, ping to `192.168.2.1` |
| **forward** | **Through** the router | VPC to `8.8.8.8` |
| **output** | **From** the router itself | R1 pinging `8.8.8.8` |

To protect R1, filter **input**. To control what the VPCs may reach, filter **forward**. Covered in section E.

</details>

## Troubleshooting scenarios

**38. A VPC runs `ip dhcp` and times out. What do you check, in order?**

<details><summary>Answer</summary>

1. **Port up?** Interfaces: the VPC's port has the **R** (running) flag. If not, check the EVE-NG link (stop/start both nodes).
2. **Port in the bridge?** Bridge → Ports: `etherX` → bridge1.
3. **Server valid?** IP → DHCP Server: interface `bridge1`, no red **I** flag.
4. **Pool linked and not full?** `address-pool=pool1`, not `static-only`; free addresses left.
5. **IP on bridge1?** IP → Addresses.
6. **Watch it live:** `/log print follow where topics~"dhcp"` while the VPC runs `ip dhcp`.

</details>

**39. `ping 8.8.8.8` works but `ping google.com` fails. What's wrong?**

<details><summary>Answer</summary>

**DNS.** Routing and NAT are fine (the IP ping works), but the name can't be turned into an IP.

Check `show ip` on the VPC (DNS empty or wrong?), the DHCP network entry's `dns-server`, and R1's own DNS (IP → DNS). After fixing the network entry, renew the VPC: `clear ip` + `ip dhcp`.

</details>

**40. `ping 192.168.1.1` works from a VPC but `ping 8.8.8.8` fails. Where's the problem?**

<details><summary>Answer</summary>

Most likely **beyond R1**. For the home router to reply to a VPC at all, the packet must have been masqueraded (the home router has no route to `192.168.2.0/24`), so R1's routing and NAT are working.

Check: can R1 itself reach the internet (`/ping 8.8.8.8` on R1)? Does R1 have a default route (`0.0.0.0/0`, DAd)? Does the home router have internet? If R1 can't ping `8.8.8.8` either, the issue is the home router or the ISP.

</details>

**41. How can you watch DHCP or NAT traffic live on R1?**

<details><summary>Answer</summary>

| Tool | 🖱️ WinBox | ⌨️ CLI | Shows |
|---|---|---|---|
| **Torch** | Tools → Torch → interface `bridge1` or `ether1` | `/tool torch interface=ether1` | Live flows: who talks to whom, how fast |
| **Sniffer** | Tools → Packet Sniffer | `/tool sniffer quick interface=bridge1 port=67,68` | Each DHCP packet (Discover, Offer, ...) |
| **Log** | Log | `/log print follow where topics~"dhcp"` | DHCP events |

Torch on **ether1** while a VPC pings `8.8.8.8` shows source `192.168.1.133`, not `192.168.2.x`: masquerade, visible.

</details>

## EVE-NG specifics

**42. Why did ether5 not exist at first?**

<details><summary>Answer</summary>

A CHR node in EVE-NG has **4** network interfaces by default (ether1–ether4), and this lab needs 5 (WAN + 4 LAN). Stop the node → right-click → **Edit** → **Ethernets** `6` → Save → start. The count can only be changed while the node is stopped.

</details>

**43. A link was added while the nodes were running, and the port has no R flag. Why?**

<details><summary>Answer</summary>

EVE-NG doesn't always attach a new link to a running node's virtual NIC. The cable shows on the canvas but the interface stays down. Stop **both** nodes on that link and start them again.

</details>

**44. What is Cloud0, and how is it different from other network types in EVE-NG?**

<details><summary>Answer</summary>

| Network | Bridges to | Use |
|---|---|---|
| **Cloud0 / Management** | `pnet0`, the EVE host's real NIC (your home LAN) | Give lab devices internet, or reach them from your PC |
| **Cloud1–Cloud9** | `pnet1`–`pnet9` | Extra host networks; only work if set up on the EVE host |
| **Bridge** (plain network) | Nothing outside the lab | A virtual switch between lab nodes only |

</details>

## MTCNA exam-style

**45.** A DHCP client gets an IP but no internet, and its gateway shows `0.0.0.0`. What's missing on the server?
a) Address pool · b) DHCP network entry · c) NAT rule · d) Default route on the router

<details><summary>Answer</summary>

**b)** The network entry supplies gateway, mask and DNS.

</details>

**46.** Which NAT action fits best when the WAN IP is assigned by DHCP?
a) src-nat · b) dst-nat · c) masquerade · d) netmap

<details><summary>Answer</summary>

**c) masquerade.** It always uses the out-interface's current IP.

</details>

**47.** ether2–ether5 are in bridge1. Where must the LAN IP address go?
a) ether2 · b) every member port · c) bridge1 · d) ether1

<details><summary>Answer</summary>

**c) bridge1.** Layer-3 config on slave ports doesn't work.

</details>

**48.** What do the flags **DAd** on `0.0.0.0/0` mean?
a) Dynamic, Active, learned from DHCP · b) Disabled, Active, default · c) Dynamic, Active, directly connected · d) Default, Added, dynamic

<details><summary>Answer</summary>

**a)** Dynamic, Active, and small **d** = from DHCP. (Capital **C** would mean connected.)

</details>

**49.** Which firewall chain filters WinBox connections **to** the router?
a) forward · b) input · c) output · d) srcnat

<details><summary>Answer</summary>

**b) input.**

</details>

**50.** A pool is `192.168.2.2-192.168.2.10`. How many clients can hold a dynamic lease at once?
a) 8 · b) 9 · c) 10 · d) 254

<details><summary>Answer</summary>

**b) 9** (.2 through .10 inclusive).

</details>

**51.** Which UDP ports does DHCP use?
a) 53 and 54 · b) 67 (server) and 68 (client) · c) 68 (server) and 67 (client) · d) 8291

<details><summary>Answer</summary>

**b)** The server listens on 67, the client on 68.

</details>

**52.** VPC1 (`192.168.2.8`) pings VPC2 (`192.168.2.9`) through bridge1. Which is true?
a) R1 routes it and NAT rewrites the source · b) R1 routes it, no NAT · c) The bridge switches it at Layer 2, no routing or NAT · d) It goes to the home router first

<details><summary>Answer</summary>

**c)** Same subnet, so it's switched.

</details>
