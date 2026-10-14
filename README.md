# MikroTik A to Z

Hands-on MikroTik RouterOS learning, from first boot to advanced routing, built and tested in an EVE-NG lab.

## Lab environment

| Item | Value |
|---|---|
| Hypervisor | VMware Workstation (bridged to wired NIC) |
| EVE-NG host | `192.168.1.131` (SSH: `ssh root@192.168.1.131`) |
| RouterOS image | MikroTik CHR 7.24.5 |
| Image path on EVE-NG | `/opt/unetlab/addons/qemu/mikrotik-7.24.5/hda.qcow2` |
| R1 | ether1 → Cloud0 (pnet0), DHCP `192.168.1.133/24` |

## Roadmap

- [x] **A. Lab setup**: EVE-NG, CHR image, first boot, home LAN access
- [ ] **B. Basics**: CLI, WinBox, users, identity, backups, upgrades
- [ ] **C. Addressing**: IP addresses, DHCP client/server, DNS
- [ ] **D. Bridging and switching**: bridges, VLANs, trunk/access ports
- [ ] **E. Firewall**: filter rules, connection tracking, address lists
- [ ] **F. NAT**: masquerade, src-nat, dst-nat / port forwarding
- [ ] **G. Static routing**: routes, distance, gateways, failover
- [ ] **H. Dynamic routing**: OSPF, BGP
- [ ] **I. Wireless and CAPsMAN**
- [ ] **J. VPN**: WireGuard, IPsec, L2TP, SSTP
- [ ] **K. QoS**: simple queues, queue trees, mangle
- [ ] **L. Hotspot and PPPoE**
- [ ] **M. Monitoring**: logging, SNMP, Netwatch, The Dude
- [ ] **N. Scripting and scheduler**
- [ ] **O. Security hardening**

## A. Lab setup

### Install CHR on EVE-NG

Run on the EVE-NG host:

```bash
mkdir -p /opt/unetlab/addons/qemu/mikrotik-7.24.5
cd /opt/unetlab/addons/qemu/mikrotik-7.24.5
wget https://download.mikrotik.com/routeros/7.24.5/chr-7.24.5.img.zip
unzip chr-7.24.5.img.zip
qemu-img convert -f raw -O qcow2 chr-7.24.5.img hda.qcow2
rm -f chr-7.24.5.img chr-7.24.5.img.zip
/opt/unetlab/wrappers/unl_wrapper -a fixpermissions
```

### First login

Username `admin`, blank password, then set a new one.

```
/system identity set name=R1
/interface print
/ip dhcp-client print
/ip address print
```

CHR ships with a DHCP client on ether1 already.

### Troubleshooting: node not reaching the LAN through Cloud0

Symptom: `/ip dhcp-client print` stays on `searching...`.

On the EVE-NG host:

```bash
brctl show pnet0          # should list eth0 AND vunl0_1_0 (R1 ether1)
ip link | grep vunl       # confirms the node's interfaces exist
```

If `vunl0_1_0` is missing from pnet0:

- Quick fix (lost on node restart): `brctl addif pnet0 vunl0_1_0`
- Proper fix: stop the node, delete the link, re-link ether1 to the Cloud0 network, and start the node.

Remember: `brctl` and `tcpdump` run on the EVE-NG host (`root@eve-ng`), not in the RouterOS console (`[admin@R1] >`).
