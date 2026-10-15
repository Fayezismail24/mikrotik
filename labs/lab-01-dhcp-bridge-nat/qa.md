# Lab 1 Q&A: DHCP Server, Bridge and NAT

> Follow-up "what if / why" questions from reviewing [Lab 1](README.md). Try each one before opening the answer.

[Lab 1](README.md) · [Labs index](../README.md) · [Back to the main index](../../README.md)

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
