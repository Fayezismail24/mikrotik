# MikroTik RouterOS v7 Cheatsheet

The most-used commands on one page. See each section folder for details.

## CLI

| Action | Command |
|---|---|
| Help / autocomplete | `?` / `Tab` |
| Safe mode | `Ctrl+X` |
| Print with detail | `print detail` / `print terse` |
| Filter | `print where interface=ether1` |
| Edit by query | `set [find name=x] comment="..."` |
| Export config | `/export compact` / `/export file=cfg show-sensitive` |
| Binary backup | `/system backup save name=bk password=xxx` |
| Restore | `/system backup load name=bk.backup` / `/import file-name=cfg.rsc` |
| Reset (empty) | `/system reset-configuration no-defaults=yes skip-backup=yes` |
| Reboot | `/system reboot` |

## System

```
/system identity set name=R1
/system clock set time-zone-name=Asia/Beirut
/system ntp client set enabled=yes servers=pool.ntp.org
/system resource print
/system package update check-for-updates
/system package update install
/user add name=fayez group=full password=xxx
/user disable admin
```

## Interfaces and addressing

```
/interface print
/ip address add address=192.168.10.1/24 interface=bridge1
/ip dhcp-client add interface=ether1 disabled=no
/ip pool add name=pool1 ranges=192.168.10.100-192.168.10.200
/ip dhcp-server network add address=192.168.10.0/24 gateway=192.168.10.1 dns-server=192.168.10.1
/ip dhcp-server add name=dhcp1 interface=bridge1 address-pool=pool1 disabled=no
/ip dns set servers=1.1.1.1,8.8.8.8 allow-remote-requests=yes
```

## Bridge and VLAN

```
/interface bridge add name=bridge1 vlan-filtering=no
/interface bridge port add bridge=bridge1 interface=ether2 pvid=10
/interface bridge vlan add bridge=bridge1 vlan-ids=10 tagged=bridge1,ether4 untagged=ether2
/interface vlan add interface=bridge1 name=vlan10 vlan-id=10
/interface bridge set bridge1 vlan-filtering=yes      # LAST
```

## Interface lists

```
/interface list add name=WAN
/interface list add name=LAN
/interface list member add list=WAN interface=ether1
/interface list member add list=LAN interface=bridge1
```

## Firewall (base)

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

## NAT

```
/ip firewall nat add chain=srcnat out-interface-list=WAN action=masquerade
/ip firewall nat add chain=dstnat in-interface-list=WAN protocol=tcp dst-port=8080 action=dst-nat to-addresses=192.168.10.50 to-ports=80
```

## Routing

```
/ip route add dst-address=0.0.0.0/0 gateway=192.168.1.1 check-gateway=ping distance=1
/ip route add dst-address=10.2.0.0/24 gateway=10.0.12.2
/ip route print where active
/ip route check 8.8.8.8
```

## OSPF (v7)

```
/routing ospf instance add name=ospf1 version=2 router-id=1.1.1.1
/routing ospf area add name=backbone area-id=0.0.0.0 instance=ospf1
/routing ospf interface-template add area=backbone interfaces=ether2 type=ptp
/routing ospf interface-template add area=backbone interfaces=ether3,lo passive
/routing ospf neighbor print
```

## BGP (v7)

```
/ip firewall address-list add list=bgp-out address=10.1.0.0/24
/routing bgp connection add name=to-R2 as=65001 local.role=ebgp remote.address=10.0.12.2 remote.as=65002 output.network=bgp-out
/routing bgp session print
```

## WireGuard

```
/interface wireguard add name=wg1 listen-port=13231
/ip address add address=10.255.0.1/30 interface=wg1
/interface wireguard peers add interface=wg1 public-key="..." endpoint-address=x.x.x.x endpoint-port=13231 allowed-address=10.255.0.2/32,192.168.20.0/24 persistent-keepalive=25s
```

## Queues

```
/queue simple add name=PC1 target=192.168.10.10/32 max-limit=2M/10M
/queue type add name=pcq-down kind=pcq pcq-classifier=dst-address
/queue type add name=pcq-up kind=pcq pcq-classifier=src-address
/queue simple add name=fair target=192.168.10.0/24 max-limit=20M/100M queue=pcq-up/pcq-down
```

## PPPoE client

```
/interface pppoe-client add name=pppoe-out1 interface=ether1 user=xxx password=xxx add-default-route=yes use-peer-dns=yes disabled=no
```

## Troubleshooting tools

```
/ping 8.8.8.8 count=4
/tool traceroute 8.8.8.8
/tool torch interface=ether1
/log print where topics~"error"
/ip firewall filter print stats
/ip firewall connection print
/interface monitor-traffic ether1
/tool sniffer quick interface=ether1
```

## Hardening (short)

```
/ip service disable telnet,ftp,www,api,api-ssl
/ip service set winbox address=192.168.1.0/24
/ip service set ssh address=192.168.1.0/24
/tool mac-server set allowed-interface-list=LAN
/tool mac-server mac-winbox set allowed-interface-list=LAN
/ip neighbor discovery-settings set discover-interface-list=LAN
/tool bandwidth-server set enabled=no
/ip ssh set strong-crypto=yes
```

## EVE-NG host (Linux, `root@eve-ng`)

```bash
brctl show pnet0                       # node attached to Cloud0?
brctl addif pnet0 vunl0_1_0            # temporary attach fix
ip link | grep vunl                    # node interfaces
tcpdump -ni pnet0 -e port 67 or port 68
/opt/unetlab/wrappers/unl_wrapper -a fixpermissions
```
