# MikroTik A to Z

A complete, hands-on MikroTik RouterOS v7 guide, from first boot to advanced routing, VPN and security. Every section is built to practice in an EVE-NG lab with MikroTik CHR.

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

WinBox paths follow WinBox 4 on RouterOS 7. Untested sections may have small menu differences until verified in the lab.

## Contents

| | Section | Topics | Status |
|---|---|---|---|
| 0 | [Fundamentals: start here](0-fundamentals/) | What a router does, MikroTik/RouterOS/CHR, interfaces, IP basics | 🔄 |
| A | [Lab setup](A-lab-setup/) | EVE-NG, CHR image, first boot, Cloud0, VPCS | ✅ |
| B | [Basics](B-basics/) | CLI, WinBox, users, NTP, backups, upgrades, reset | 🔄 |
| C | [Addressing](C-addressing/) | IP, DHCP client/server, DNS, ARP | 📝 |
| D | [Bridging and VLANs](D-bridging-vlans/) | Bridge, VLAN filtering, access/trunk, inter-VLAN | 📝 |
| E | [Firewall](E-firewall/) | Chains, conntrack, base ruleset, address lists, brute-force | 📝 |
| F | [NAT](F-nat/) | Masquerade, port forwarding, hairpin, netmap | 📝 |
| G | [Static routing](G-static-routing/) | Routes, failover, recursive, ECMP, PBR | 📝 |
| H | [Dynamic routing](H-dynamic-routing/) | OSPF, BGP (v7 syntax), routing filters | 📝 |
| I | [Wireless and CAPsMAN](I-wireless-capsman/) | wifi vs. wireless packages, AP, CAPsMAN | 📝 (needs hardware) |
| J | [VPN](J-vpn/) | WireGuard, IPsec IKEv2, L2TP/IPsec, SSTP | 📝 |
| K | [QoS](K-qos/) | Simple queues, PCQ, mangle + queue tree, CAKE | 📝 |
| L | [Hotspot and PPPoE](L-hotspot-pppoe/) | Captive portal, User Manager, PPPoE server/client | 📝 |
| M | [Monitoring](M-monitoring/) | Logs, torch, sniffer, Netwatch, SNMP, The Dude | 📝 |
| N | [Scripting](N-scripting/) | Language basics, scheduler, backup scripts, REST API | 📝 |
| O | [Security hardening](O-security-hardening/) | Services, users, MAC access, device-mode, audit | 📝 |

📄 **[Cheatsheet](cheatsheet.md)**: the most-used tasks on one page, each with its WinBox path and CLI command.

## Lab environment

| Item | Value |
|---|---|
| Hypervisor | VMware Workstation (bridged to wired NIC) |
| EVE-NG host | `192.168.1.131` |
| RouterOS | CHR 7.24.5 |
| Home LAN ("internet" for labs) | `192.168.1.0/24` via Cloud0 / pnet0 |

## Suggested learning order

0 → A → B → C → E → F → D → G → J → H → K → M → N → O → L → I

The firewall and NAT come early because almost every later lab needs them.

## References

- MikroTik documentation: https://help.mikrotik.com/docs/
- MikroTik forum: https://forum.mikrotik.com/
- Downloads (RouterOS, CHR, WinBox): https://mikrotik.com/download
