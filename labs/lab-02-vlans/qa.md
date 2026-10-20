# Lab 2 Q&A: VLANs

> Follow-up "what if / why" questions raised while doing [Lab 2](README.md). Try each one before opening the answer.

[Lab 2](README.md) · [Labs index](../README.md) · [Back to the main index](../../README.md)

## Contents

| Section | Questions |
|---|---|
| [VLAN basics](#vlan-basics) | 1–2 |
| [PVID and access ports](#pvid-and-access-ports) | 3–4 |
| [Lab setup](#lab-setup) | 5–7 |

---

## VLAN basics

**1. If VLAN filtering is off on the bridge, are all VLANs one?**

<details><summary>Answer</summary>

Yes. With `vlan-filtering=no` the bridge ignores the PVIDs and the VLAN table. Every port behaves as if it's in one big network, and tagged frames pass through unchecked. That's why Lab 1's bridge1 showed PVID 1 everywhere and still worked as one LAN. The VLAN settings only take effect once filtering is turned on.

</details>

**2. Do VLANs remove broadcasts?**

<details><summary>Answer</summary>

No, they **contain** them. A PC still sends ARP broadcasts, but the broadcast stays inside its own VLAN. Each VLAN is its own broadcast domain, so a broadcast from VPC2 (VLAN 20) only comes out of the other VLAN 20 port (ether4), never the VLAN 30 or 40 ports.

</details>

## PVID and access ports

**3. A port has `pvid=10` but is untagged in VLAN 20. What goes wrong?**

<details><summary>Answer</summary>

The PC's plain frames become VLAN 10 on the way in, but only VLAN 20 may leave the port. Replies (in VLAN 10) can't come back out, so it's a one-way street and the PC reaches nobody. With ingress filtering on (default), the switch even drops the frames on the way in, because the port isn't a member of VLAN 10.

Rule: on an access port, **PVID = untagged VLAN**.

</details>

**4. On Cisco, an access port in VLAN 20 drops a frame tagged VLAN 10. Is MikroTik the same? And why can a PVID mismatch happen on MikroTik but not on Cisco?**

<details><summary>Answer</summary>

A tagged frame of the wrong VLAN is dropped on both. On MikroTik it's dropped by ingress filtering (port not a member of VLAN 10) or by `frame-types=admit-only-untagged-and-priority-tagged`.

The mismatch is different: a PC never tags, so a mismatch isn't caused by the PC. It's a switch config error.

| | Cisco | MikroTik |
|---|---|---|
| Access port config | `switchport mode access` + `switchport access vlan 20` | `pvid=20` on the port **and** `untagged=` in VLAN 20 |
| In and out VLAN | One command sets both | Two separate settings |
| Can they mismatch? | No | Yes, keep them equal yourself |

</details>

## Lab setup

**5. Why set the system identity?**

<details><summary>Answer</summary>

It's optional; VLANs work the same whatever the router is called. But every fresh CHR is named `CHR`, so the name tells consoles apart (`[admin@SW1]` vs `[admin@R1]`). It also shows in WinBox Neighbors, in exports/backups, and in logs and monitoring, which matters when a real network has dozens of devices.

</details>

**6. Why does SW1 have its own uplink on ether1?**

<details><summary>Answer</summary>

It's **out-of-band management**: a port only for managing the switch, outside bridge1. If the VLAN config breaks, WebFig/WinBox still reach SW1 through ether1. It's also why turning VLAN filtering on was safe: on a switch managed *through* the bridge, that step is where people lock themselves out.

</details>

**7. How do you check the IP address on a VPC?**

<details><summary>Answer</summary>

`show ip` in the VPCS console. It shows IP/MASK, GATEWAY and the MAC (`00:50:79:66:68:xx`). `show` gives a one-line summary and `?` lists all commands. Set an address with `ip <ip>/<mask> <gateway>` and keep it with `save`.

</details>

---

[← Lab 2](README.md)
