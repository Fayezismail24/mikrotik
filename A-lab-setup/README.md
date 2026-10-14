# A. Lab Setup (EVE-NG + MikroTik CHR)

> **Status:** ✅ Tested in lab (CHR 7.24.5, EVE-NG Community on VMware Workstation)

## Concept

**CHR (Cloud Hosted Router)** is RouterOS built for virtual machines. It runs the same RouterOS as a physical MikroTik, minus hardware features like wireless chips and switch chips, so it's the right image for learning in EVE-NG.

**CHR licensing:** the free license limits each interface to **1 Mbit/s upload**. That's fine for labs, since config and routing behave the same. A free 60-day trial (P1/P-Unlimited) removes the limit if you need to test throughput.

**EVE-NG networking:**
- **Cloud0 / pnet0** is bridged to the EVE-NG host's NIC, which puts nodes on your real home LAN.
- **Cloud1–9 / pnet1–9** can be bridged to extra host NICs.
- A **Bridge** network is an internal-only switch between lab nodes.
- **VPCS** (Virtual PC) is a lightweight test PC, useful as a LAN client in labs.

## Lab environment

| Item | Value |
|---|---|
| Hypervisor | VMware Workstation, bridged to wired NIC |
| EVE-NG host | `192.168.1.131` (`ssh root@192.168.1.131`) |
| Home LAN | `192.168.1.0/24` |
| RouterOS image | CHR 7.24.5 |
| R1 | ether1 → Cloud0 (pnet0), DHCP `192.168.1.133/24` |

## 1. Install the CHR image on EVE-NG

Check the latest stable version at mikrotik.com/download, then run on the EVE-NG host:

```bash
VER=7.24.5
mkdir -p /opt/unetlab/addons/qemu/mikrotik-$VER
cd /opt/unetlab/addons/qemu/mikrotik-$VER
wget https://download.mikrotik.com/routeros/$VER/chr-$VER.img.zip
unzip chr-$VER.img.zip
qemu-img convert -f raw -O qcow2 chr-$VER.img hda.qcow2
rm -f chr-$VER.img chr-$VER.img.zip
/opt/unetlab/wrappers/unl_wrapper -a fixpermissions
ls -lh        # only hda.qcow2 should remain
```

Rules EVE-NG enforces:
- The folder name must start with `mikrotik-`.
- The disk must be named `hda.qcow2`.

## 2. Copy/paste-friendly access

- **EVE-NG host:** use SSH from Windows (`ssh root@<eve-ip>`) instead of the VM console, which has no clipboard.
- **Node consoles:** use the **Native console** (PuTTY) at web UI login. Right-click pastes.

## 3. Create the first lab

1. Web UI → **Add new lab**.
2. Right-click the canvas → **Node** → **MikroTik RouterOS**, then pick the image and set the number of nodes.
3. Right-click → **Network** → type **Management(Cloud0)** to reach your home LAN.
4. **Stop the node before adding links.** Drag from the node to the network and pick `ether1`.
5. Start the node and click it to open the console.

## 4. First login

- Username `admin`, blank password.
- Decline viewing the license (`n`), then set a new password.

### Set the identity

🖱️ **WinBox:** System → Identity → `R1` → OK

⌨️ **CLI:**
```
/system identity set name=R1
```

### Check interfaces and DHCP

CHR ships with a DHCP client already on ether1.

🖱️ **WinBox:**
- **Interfaces**: ether1–ether4 are listed.
- IP → **DHCP Client**: Status should be `bound`.
- IP → **Addresses**: a `192.168.1.x` address with flag **D**.

⌨️ **CLI:**
```
/interface print
/ip dhcp-client print
/ip address print
```

### Connect with WinBox

Once ether1 has an IP:
1. Open WinBox.
2. Type `192.168.1.133` (or pick R1 on the **Neighbors** tab).
3. Log in.

## 5. Adding a test PC (VPCS)

Right-click → **Node** → **Virtual PC (VPCS)**. Its console commands:

```
ip 192.168.10.10/24 192.168.10.1     # static IP + gateway
ip dhcp                               # or get one by DHCP
show ip
ping 192.168.10.1
save
```

## Verify

| Check | 🖱️ WinBox | ⌨️ CLI |
|---|---|---|
| DHCP bound | IP → DHCP Client (Status `bound`) | `/ip dhcp-client print` |
| IP on ether1 | IP → Addresses | `/ip address print` |
| Gateway reachable | Tools → Ping → `192.168.1.1` | `/ping 192.168.1.1 count=4` |
| Internet reachable | Tools → Ping → `8.8.8.8` | `/ping 8.8.8.8 count=4` |

The EVE-NG install and troubleshooting steps below are Linux commands on the EVE-NG host, so they're CLI only.

## Troubleshooting

### DHCP stuck on `searching...`

On the **EVE-NG host** (`root@eve-ng`), not in RouterOS:

```bash
brctl show pnet0            # must list eth0 AND vunl0_<node>_0
ip link | grep vunl         # node interfaces exist?
tcpdump -ni pnet0 -e port 67 or port 68
```

Interface naming: `vunl0_1_0` means user 0, node 1, interface 0 (ether1).

| Finding | Fix |
|---|---|
| vunl not in pnet0 | Quick fix: `brctl addif pnet0 vunl0_1_0` (lost on restart). Proper fix: stop node, delete link, re-link, start. |
| Discover leaves, no Offer | VMware: bridge VMnet0 to the **wired** NIC, not Wi-Fi. ESXi: Promiscuous / MAC changes / Forged transmits = Accept. Hyper-V: enable MAC spoofing. |
| Nothing in tcpdump | The node isn't attached. See the row above. |

### Other gotchas

- `bad command name brctl` means you typed a Linux command in the RouterOS console. Check your prompt: `[admin@R1] >` is RouterOS, `root@eve-ng:~#` is the host.
- `router was rebooted without proper shutdown` comes from **Stop** in EVE-NG, which is a hard power-off. It's harmless.
- `failure: dhcp-client on that interface already exists` means CHR already has one on ether1.
