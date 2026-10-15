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
| [DHCP: leases](#dhcp-leases) | 17–18 |
| [NAT](#nat) | 19–22 |
| [Default route](#default-route) | 23 |
| [Troubleshooting scenarios](#troubleshooting-scenarios) | 24–26 |
| [EVE-NG specifics](#eve-ng-specifics) | 27–28 |
| [MTCNA exam-style](#mtcna-exam-style) | 29–33 |

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

Fix: make the pool bigger, e.g. `/ip pool set pool1 ranges=192.168.2.2-192.168.2.254`.

</details>

**13. Pool vs. network entry: what is the difference?**

<details><summary>Answer</summary>

| | Pool | Network entry |
|---|---|---|
| Answers | **Which IP** does the client get? | **What else** does it need? |
| Contains | A range: `192.168.2.2-192.168.2.10` | Subnet, gateway, DNS: `192.168.2.0/24`, `.2.1`, `8.8.8.8` |
| Linked by | `address-pool=pool1` on the server | Matched automatically by subnet |
| Missing → | Server has no addresses to hand out (`static-only`) | Client gets an IP with `/0` mask and gateway `0.0.0.0` |

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

## DHCP: leases

**17. Where do you see which PC got which IP?**

<details><summary>Answer</summary>

🖱️ IP → DHCP Server → **Leases**
⌨️ `/ip dhcp-server lease print`

Each row shows address, MAC, hostname and status. **D** = dynamic (given from the pool). Status **bound** = in use. Expect one lease per VPC.

</details>

**18. The DHCP server shows `address-pool=static-only`. Why do the VPCs get nothing?**

<details><summary>Answer</summary>

`static-only` means no pool is linked, so the server has no addresses to hand out to new clients. That was a real mistake in this lab. Fix: `/ip dhcp-server set server1 address-pool=pool1`.

</details>

## NAT

**19. What exactly does masquerade change in the packet?**

<details><summary>Answer</summary>

On the way **out** it replaces the **source IP** (`192.168.2.8`) with ether1's IP (`192.168.1.133`), and changes the source port if needed. Connection tracking remembers the swap. On the way **back** the reply is addressed to `192.168.1.133`; R1 looks it up and rewrites the destination back to `192.168.2.8`.

```
Out:  192.168.2.8:50000 → 8.8.8.8    becomes  192.168.1.133:50000 → 8.8.8.8
Back: 8.8.8.8 → 192.168.1.133:50000  becomes  8.8.8.8 → 192.168.2.8:50000
```

</details>

**20. Why set `out-interface=ether1` on the NAT rule?**

<details><summary>Answer</summary>

So NAT only touches traffic **going out to the home LAN / internet**. That's the only place the private `192.168.2.x` addresses need hiding, because the home router doesn't know that subnet.

</details>

**21. When VPC1 pings VPC2, does the packet go through NAT?**

<details><summary>Answer</summary>

No. VPC1 to VPC2 is switched inside bridge1 at Layer 2, so it never reaches R1's routing or NAT. And the NAT rule only matches traffic leaving via ether1 anyway. The NAT counter doesn't move during that ping.

</details>

**22. How do you see NAT working?**

<details><summary>Answer</summary>

Watch the rule's counters while a VPC pings `8.8.8.8`: they rise.

🖱️ IP → Firewall → **NAT** (Bytes / Packets columns)
⌨️ `/ip firewall nat print stats`

</details>

## Default route

**23. Where does R1's default route come from, and what if it's missing?**

<details><summary>Answer</summary>

The DHCP client on ether1 has `add-default-route=yes` (the default), so it installs `0.0.0.0/0 via 192.168.1.1` with flags **DAd** (d = from DHCP).

If it's missing (e.g. `add-default-route=no`): R1 and the VPCs still reach `192.168.1.x` (connected route), but `8.8.8.8` fails because R1 has no route to it.

🖱️ IP → DHCP Client → ether1 → **Add Default Route** = yes
⌨️ `/ip dhcp-client set [find interface=ether1] add-default-route=yes`

</details>

## Troubleshooting scenarios

**24. A VPC runs `ip dhcp` and times out. What do you check, in order?**

<details><summary>Answer</summary>

1. **Port up?** Interfaces: the VPC's port has the **R** (running) flag. If not, check the EVE-NG link (stop/start both nodes).
2. **Port in the bridge?** Bridge → Ports: `etherX` → bridge1.
3. **Server valid?** IP → DHCP Server: interface `bridge1`, no red **I** flag.
4. **Pool linked and not full?** `address-pool=pool1`, not `static-only`; free addresses left.
5. **IP on bridge1?** IP → Addresses.
6. **Watch it live:** `/log print follow where topics~"dhcp"` while the VPC runs `ip dhcp`.

</details>

**25. `ping 8.8.8.8` works but `ping google.com` fails. What's wrong?**

<details><summary>Answer</summary>

**DNS.** Routing and NAT are fine (the IP ping works), but the name can't be turned into an IP.

Check `show ip` on the VPC (DNS empty or wrong?), the DHCP network entry's `dns-server`, and R1's own DNS (IP → DNS). After fixing the network entry, renew the VPC: `clear ip` + `ip dhcp`.

</details>

**26. `ping 192.168.1.1` works from a VPC but `ping 8.8.8.8` fails. Where's the problem?**

<details><summary>Answer</summary>

Most likely **beyond R1**. For the home router to reply to a VPC at all, the packet must have been masqueraded (the home router has no route to `192.168.2.0/24`), so R1's routing and NAT are working.

Check: can R1 itself reach the internet (`/ping 8.8.8.8` on R1)? Does R1 have a default route (`0.0.0.0/0`, DAd)? Does the home router have internet? If R1 can't ping `8.8.8.8` either, the issue is the home router or the ISP.

</details>

## EVE-NG specifics

**27. Why did ether5 not exist at first?**

<details><summary>Answer</summary>

A CHR node in EVE-NG has **4** network interfaces by default (ether1–ether4), and this lab needs 5 (WAN + 4 LAN). Stop the node → right-click → **Edit** → **Ethernets** `6` → Save → start. The count can only be changed while the node is stopped.

</details>

**28. A link was added while the nodes were running, and the port has no R flag. Why?**

<details><summary>Answer</summary>

EVE-NG doesn't always attach a new link to a running node's virtual NIC. The cable shows on the canvas but the interface stays down. Stop **both** nodes on that link and start them again.

</details>

## MTCNA exam-style

**29.** A DHCP client gets an IP but no internet, and its gateway shows `0.0.0.0`. What's missing on the server?
a) Address pool · b) DHCP network entry · c) NAT rule · d) Default route on the router

<details><summary>Answer</summary>

**b)** The network entry supplies gateway, mask and DNS.

</details>

**30.** ether2–ether5 are in bridge1. Where must the LAN IP address go?
a) ether2 · b) every member port · c) bridge1 · d) ether1

<details><summary>Answer</summary>

**c) bridge1.** Layer-3 config on slave ports doesn't work.

</details>

**31.** What do the flags **DAd** on `0.0.0.0/0` mean?
a) Dynamic, Active, learned from DHCP · b) Disabled, Active, default · c) Dynamic, Active, directly connected · d) Default, Added, dynamic

<details><summary>Answer</summary>

**a)** Dynamic, Active, and small **d** = from DHCP. (Capital **C** would mean connected.)

</details>

**32.** A pool is `192.168.2.2-192.168.2.10`. How many clients can hold a dynamic lease at once?
a) 8 · b) 9 · c) 10 · d) 254

<details><summary>Answer</summary>

**b) 9** (.2 through .10 inclusive).

</details>

**33.** VPC1 (`192.168.2.8`) pings VPC2 (`192.168.2.9`) through bridge1. Which is true?
a) R1 routes it and NAT rewrites the source · b) R1 routes it, no NAT · c) The bridge switches it at Layer 2, no routing or NAT · d) It goes to the home router first

<details><summary>Answer</summary>

**c)** Same subnet, so it's switched.

</details>
