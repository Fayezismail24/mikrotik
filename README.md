# MikroTik A to Z

A complete, hands-on MikroTik RouterOS v7 guide, from first boot to advanced routing, VPN and security. Every section is built to practice in an EVE-NG lab with MikroTik CHR, plus a physical hAP for the hardware and Wi-Fi topics.

**Goal of this repo:** redo **MTCNA** hands-on with a deep understanding of every topic, then prepare for **MTCRE**.

## How each section is organized

1. **Concept:** what it is, why it exists, and how it differs from similar things (with analogies and comparison tables).
2. **Config steps:** every step shows **both** ways to do it:
   - 🖱️ **WinBox:** the menu path and the fields to fill.
   - ⌨️ **CLI:** the RouterOS v7 command, ready to paste.
3. **EVE-NG lab:** a topology and step-by-step exercise.
4. **Verify:** a table of where to look in WinBox and which `print` command to run.
5. **Troubleshooting:** common mistakes and fixes.

Example:

> 🖱️ **WinBox:** IP → Addresses → **+** → Address `192.168.10.1/24`, Interface `ether2` → OK
>
> ⌨️ **CLI:** `/ip address add address=192.168.10.1/24 interface=ether2`

**Status:** ✅ Tested in lab · 🔄 Partly tested · 📝 Written, not yet tested

**Cert tags:** each section starts with the MikroTik certification it belongs to. 🔹 marks content beyond MTCNA/MTCRE.

WinBox paths follow WinBox 4 on RouterOS 7. Untested sections may have small menu differences until verified in the lab.

## Contents

| | Section | Topics | Cert | Status |
|---|---|---|---|---|
| 0 | [Fundamentals: start here](0-fundamentals/) | One file per lesson: router basics, MikroTik/RouterOS/CHR, interfaces, network devices (hub/switch/L3 switch/router/AP/modem/firewall), MikroTik product family, IP basics | MTCNA | 🔄 |
| A | [Lab setup](A-lab-setup/) | EVE-NG, CHR image, first boot, Cloud0, VPCS | Lab | ✅ |
| B | [Basics](B-basics/) | CLI, WinBox, users, NTP, backups, upgrades, default config, QuickSet, Netinstall, licenses, firmware | MTCNA | 🔄 |
| C | [Addressing](C-addressing/) | IP, DHCP client/server, DNS, ARP | MTCNA | 📝 |
| D | [Bridging and VLANs](D-bridging-vlans/) | Bridge, VLAN filtering, access/trunk, inter-VLAN | MTCNA · MTCRE | 📝 |
| E | [Firewall](E-firewall/) | Chains, conntrack, base ruleset, address lists, brute-force | MTCNA · MTCSE | 📝 |
| F | [NAT](F-nat/) | Masquerade, port forwarding, hairpin, netmap | MTCNA | 📝 |
| G | [Static routing](G-static-routing/) | Routes, failover, scope/target-scope, /32 addressing, ECMP, PBR, VRRP | MTCNA · MTCRE | 📝 |
| H | [Dynamic routing](H-dynamic-routing/) | OSPF single/multi-area, LSA types, stub/NSSA, redistribution · 🔹 BGP | MTCRE · 🔹 MTCINE | 📝 |
| I | [Wireless and CAPsMAN](I-wireless-capsman/) | wifi vs. wireless packages, AP, CAPsMAN | MTCNA · MTCWE | 📝 (hAP available) |
| J | [VPN and tunnels](J-vpn/) | WireGuard, IPsec IKEv2, L2TP/IPsec, SSTP, EoIP, IPIP, GRE | MTCNA · MTCRE · MTCSE | 📝 |
| K | [QoS](K-qos/) | Simple queues, PCQ, mangle + queue tree · 🔹 CAKE | MTCNA · MTCTCE | 📝 |
| L | [Hotspot and PPPoE](L-hotspot-pppoe/) | Captive portal, User Manager, PPPoE server/client | MTCNA · MTCUME | 📝 |
| M | [Monitoring](M-monitoring/) | Logs, torch, sniffer, Netwatch, SNMP, The Dude, supout.rif | MTCNA | 📝 |
| N | [Scripting](N-scripting/) | Language basics, scheduler, backup scripts, REST API | 🔹 Beyond | 📝 |
| O | [Security hardening](O-security-hardening/) | Services, users, MAC access, device-mode, audit | MTCSE | 📝 |
| 🧪 | [Labs](labs/) | Complete hands-on EVE-NG exercises combining several sections. Lab 1: DHCP server, bridge and NAT | MTCNA | 🔄 |
| 🧠 | [Q&A](qa/) | Follow-up "what if / why" questions and answers for each lab | MTCNA | 🔄 |

📄 **[Cheatsheet](cheatsheet.md)**: the most-used tasks on one page, each with its WinBox path and CLI command.

## Certification paths

### MTCNA: MikroTik Certified Network Associate ✅ passed, redoing hands-on

| Module | Topics | Repo |
|---|---|---|
| 1. Introduction | RouterOS, RouterBOARD, first access, WinBox, default config, QuickSet, upgrades, firmware, Netinstall, licenses, backup/export, reset | [0](0-fundamentals/), [A](A-lab-setup/), [B](B-basics/) |
| 2. DHCP | DHCP server/client, leases, static leases, ARP | [C](C-addressing/) |
| 3. Bridging | Bridges, bridge ports, wireless in a bridge | [D](D-bridging-vlans/) §1 |
| 4. Routing | Static routes, default route, gateway check, distance | [G](G-static-routing/) §1–2 |
| 5. Wireless | Wi-Fi concepts, AP setup, security, access list, registration table | [I](I-wireless-capsman/) |
| 6. Firewall | Filter chains, connection states, address lists, NAT (src/dst) | [E](E-firewall/), [F](F-nat/) |
| 7. QoS | Simple queues, burst, PCQ | [K](K-qos/) §1–2 |
| 8. Tunnels | PPP, PPPoE, IP pools, secrets/profiles, EoIP, VPN | [J](J-vpn/), [L](L-hotspot-pppoe/) Part 2–3 |
| 9. Tools | Email, Netwatch, ping, traceroute, torch, graphs, SNMP, The Dude, logs, supout.rif | [M](M-monitoring/) |

### MTCRE: MikroTik Certified Routing Engineer 🎯 next goal

| Topic | Repo |
|---|---|
| Static routing in depth: scope / target-scope, recursive routes, ECMP, policy routing | [G](G-static-routing/) §2–5 |
| Point-to-point /32 addressing | [G](G-static-routing/) §3b |
| VRRP | [G](G-static-routing/) §7 |
| OSPF: single and multi-area, LSA types, stub/NSSA, cost, redistribution, default route | [H](H-dynamic-routing/) Part 1 + 1b |
| VLANs | [D](D-bridging-vlans/) |
| Tunnels: EoIP, IPIP, GRE, PPP-based VPNs | [J](J-vpn/) |

### Other MikroTik certs (for later)

| Cert | Focus | Repo |
|---|---|---|
| MTCSE | Security | [E](E-firewall/), [J](J-vpn/), [O](O-security-hardening/) |
| MTCWE | Wireless | [I](I-wireless-capsman/) |
| MTCTCE | Traffic control / QoS | [K](K-qos/) |
| MTCUME | User management (Hotspot, User Manager, RADIUS) | [L](L-hotspot-pppoe/) |
| MTCINE | Inter-networking: BGP, MPLS (needs MTCRE) | [H](H-dynamic-routing/) Part 2 |

## Lab environment

| Item | Value |
|---|---|
| Hypervisor | VMware Workstation (bridged to wired NIC) |
| EVE-NG host | `192.168.1.131` |
| RouterOS | CHR 7.24.5 |
| Home LAN ("internet" for labs) | `192.168.1.0/24` via Cloud0 / pnet0 |
| Physical hardware | Spare MikroTik hAP (model TBD), bought for MTCNA. It's not the home router, so it's free to reset. Used for B (default config, Netinstall, firmware) and [I](I-wireless-capsman/). |

## Suggested learning order

0 → A → B → C → E → F → D → G → J → H → K → M → N → O → L → I

The firewall and NAT come early because almost every later lab needs them.

## References

- MikroTik documentation: https://help.mikrotik.com/docs/
- MikroTik training and certification outlines: https://mikrotik.com/training
- MikroTik forum: https://forum.mikrotik.com/
- Downloads (RouterOS, CHR, WinBox, Netinstall): https://mikrotik.com/download
