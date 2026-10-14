# MikroTik RouterOS v7 Cheatsheet

The most-used tasks, each with its 🖱️ WinBox path and ⌨️ CLI command. See each section folder for full steps.

## General

| Task | 🖱️ WinBox | ⌨️ CLI |
|---|---|---|
| Open the CLI | **New Terminal** | (SSH / console) |
| Safe Mode | **Safe Mode** button (top-left) | `Ctrl+X` |
| Router info | System → Resources | `/system resource print` |
| Identity | System → Identity | `/system identity set name=R1` |
| Time zone | System → Clock | `/system clock set time-zone-name=Asia/Beirut` |
| NTP | System → NTP Client | `/system ntp client set enabled=yes servers=pool.ntp.org` |
| Add user | System → Users → + | `/user add name=me group=full password=xxx` |
| Disable admin | System → Users → admin → ✕ | `/user disable admin` |
| Binary backup | Files → Backup | `/system backup save name=bk password=xxx` |
| Restore backup | Files → select → Restore | `/system backup load name=bk.backup` |
| Export config | (New Terminal) | `/export file=cfg show-sensitive` |
| Import config | Files + New Terminal | `/import file-name=cfg.rsc` |
| Update RouterOS | System → Packages → Check For Updates | `/system package update install` |
| Reset (empty) | System → Reset Configuration → No Default Config | `/system reset-configuration no-defaults=yes skip-backup=yes` |
| Reboot | System → Reboot | `/system reboot` |

## Addressing

| Task | 🖱️ WinBox | ⌨️ CLI |
|---|---|---|
| Add IP | IP → Addresses → + | `/ip address add address=192.168.10.1/24 interface=bridge1` |
| DHCP client | IP → DHCP Client → + | `/ip dhcp-client add interface=ether1 disabled=no` |
| DHCP server | IP → DHCP Server → DHCP Setup | `/ip dhcp-server setup` |
| Leases | IP → DHCP Server → Leases | `/ip dhcp-server lease print` |
| DNS | IP → DNS | `/ip dns set servers=1.1.1.1 allow-remote-requests=yes` |
| Static DNS | IP → DNS → Static | `/ip dns static add name=r1.lab address=192.168.10.1` |

## Bridge and VLAN

| Task | 🖱️ WinBox | ⌨️ CLI |
|---|---|---|
| Bridge | Bridge → + | `/interface bridge add name=bridge1 vlan-filtering=no` |
| Access port | Bridge → Ports → + (VLAN tab: PVID) | `/interface bridge port add bridge=bridge1 interface=ether2 pvid=10` |
| VLAN table | Bridge → VLANs → + | `/interface bridge vlan add bridge=bridge1 vlan-ids=10 tagged=bridge1,ether4 untagged=ether2` |
| VLAN interface | Interfaces → + → VLAN | `/interface vlan add interface=bridge1 name=vlan10 vlan-id=10` |
| Enable filtering (LAST) | Bridge → bridge1 → VLAN tab ✓ | `/interface bridge set bridge1 vlan-filtering=yes` |

## Interface lists

| Task | 🖱️ WinBox | ⌨️ CLI |
|---|---|---|
| Create list | Interfaces → Interface List → Lists → + | `/interface list add name=WAN` |
| Add member | Interfaces → Interface List → + | `/interface list member add list=WAN interface=ether1` |

## Firewall and NAT (IP → Firewall)

| Task | 🖱️ WinBox | ⌨️ CLI |
|---|---|---|
| Filter rule | Filter Rules → + (General / Advanced / Action tabs) | `/ip firewall filter add chain=input ...` |
| Move a rule | drag and drop | `place-before=` |
| Masquerade | NAT → + → srcnat, Out. Int. List WAN, masquerade | `/ip firewall nat add chain=srcnat out-interface-list=WAN action=masquerade` |
| Port forward | NAT → + → dstnat, tcp, Dst. Port, Action dst-nat | `/ip firewall nat add chain=dstnat in-interface-list=WAN protocol=tcp dst-port=8080 action=dst-nat to-addresses=192.168.10.50 to-ports=80` |
| Address list | Address Lists → + | `/ip firewall address-list add list=X address=1.2.3.4` |
| Connections | Connections tab | `/ip firewall connection print` |
| Counters | Bytes / Packets columns | `/ip firewall filter print stats` |

Base firewall (paste in New Terminal):
```
/ip firewall filter
add chain=input action=accept connection-state=established,related,untracked
add chain=input action=drop connection-state=invalid
add chain=input action=accept protocol=icmp
add chain=input action=drop in-interface-list=!LAN comment="IN: drop all not from LAN"
add chain=forward action=fasttrack-connection connection-state=established,related hw-offload=yes
add chain=forward action=accept connection-state=established,related,untracked
add chain=forward action=drop connection-state=invalid
add chain=forward action=drop connection-state=new connection-nat-state=!dstnat in-interface-list=WAN
```

## Routing

| Task | 🖱️ WinBox | ⌨️ CLI |
|---|---|---|
| Static route | IP → Routes → + | `/ip route add dst-address=10.2.0.0/24 gateway=10.0.12.2` |
| Default + failover | IP → Routes → + (Check Gateway, Distance) | `/ip route add dst-address=0.0.0.0/0 gateway=192.168.1.1 check-gateway=ping distance=1` |
| Routing table | Routing → Tables → + | `/routing table add name=via-isp2 fib` |
| Routing rule | Routing → Rules → + | `/routing rule add src-address=... action=lookup-only-in-table table=via-isp2` |
| OSPF instance | Routing → OSPF → Instances | `/routing ospf instance add name=ospf1 version=2 router-id=1.1.1.1` |
| OSPF area | Routing → OSPF → Areas | `/routing ospf area add name=backbone area-id=0.0.0.0 instance=ospf1` |
| OSPF interfaces | Routing → OSPF → Interface Templates | `/routing ospf interface-template add area=backbone interfaces=ether2 type=ptp` |
| OSPF neighbors | Routing → OSPF → Neighbors | `/routing ospf neighbor print` |
| BGP peer | Routing → BGP → Connections | `/routing bgp connection add name=to-R2 as=65001 local.role=ebgp remote.address=10.0.12.2 remote.as=65002 output.network=bgp-out` |
| BGP status | Routing → BGP → Sessions | `/routing bgp session print` |

## VPN

| Task | 🖱️ WinBox | ⌨️ CLI |
|---|---|---|
| WireGuard interface | WireGuard → + | `/interface wireguard add name=wg1 listen-port=13231` |
| WireGuard peer | WireGuard → Peers → + | `/interface wireguard peers add interface=wg1 public-key="..." endpoint-address=x.x.x.x endpoint-port=13231 allowed-address=...` |
| IPsec status | IP → IPsec → Active Peers / Installed SAs | `/ip ipsec active-peers print` |
| L2TP server | PPP → L2TP Server | `/interface l2tp-server server set enabled=yes use-ipsec=required ...` |
| VPN users | PPP → Secrets | `/ppp secret add name=u password=p service=l2tp` |
| Online VPN users | PPP → Active Connections | `/ppp active print` |

## Queues

| Task | 🖱️ WinBox | ⌨️ CLI |
|---|---|---|
| Limit one IP | Queues → Simple Queues → + | `/queue simple add name=PC1 target=192.168.10.10/32 max-limit=2M/10M` |
| PCQ type | Queues → Queue Types → + | `/queue type add name=pcq-down kind=pcq pcq-classifier=dst-address` |
| Disable fasttrack | IP → Firewall → fasttrack rule → ✕ | `/ip firewall filter disable [find action=fasttrack-connection]` |

## PPPoE / Hotspot

| Task | 🖱️ WinBox | ⌨️ CLI |
|---|---|---|
| PPPoE client | PPP → + → PPPoE Client | `/interface pppoe-client add name=pppoe-out1 interface=ether1 user=x password=x add-default-route=yes disabled=no` |
| PPPoE server | PPP → PPPoE Servers → + | `/interface pppoe-server server add service-name=ISP interface=ether2 disabled=no` |
| Hotspot | IP → Hotspot → Hotspot Setup | `/ip hotspot setup` |

## Troubleshooting

| Task | 🖱️ WinBox | ⌨️ CLI |
|---|---|---|
| Ping | Tools → Ping | `/ping 8.8.8.8 count=4` |
| Traceroute | Tools → Traceroute | `/tool traceroute 8.8.8.8` |
| Live traffic | Tools → Torch | `/tool torch interface=ether1` |
| Capture | Tools → Packet Sniffer | `/tool sniffer quick interface=ether1` |
| Logs | Log | `/log print where topics~"error"` |
| Interface traffic | Interfaces (double-click → Traffic) | `/interface monitor-traffic ether1` |

## Hardening

| Task | 🖱️ WinBox | ⌨️ CLI |
|---|---|---|
| Disable services | IP → Services | `/ip service disable telnet,ftp,www,api,api-ssl` |
| Restrict WinBox/SSH | IP → Services → Available From | `/ip service set winbox address=192.168.1.0/24` |
| MAC access | Tools → MAC Server | `/tool mac-server mac-winbox set allowed-interface-list=LAN` |
| Discovery | IP → Neighbors → Discovery Settings | `/ip neighbor discovery-settings set discover-interface-list=LAN` |
| Strong SSH | IP → SSH | `/ip ssh set strong-crypto=yes` |

## EVE-NG host (Linux, `root@eve-ng`, no GUI)

```bash
brctl show pnet0                       # node attached to Cloud0?
brctl addif pnet0 vunl0_1_0            # temporary attach fix
ip link | grep vunl                    # node interfaces
tcpdump -ni pnet0 -e port 67 or port 68
/opt/unetlab/wrappers/unl_wrapper -a fixpermissions
```
