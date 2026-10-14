# E. Firewall

> **Status:** 📝 Not yet tested in lab

## Concept

The RouterOS firewall is **stateful**. It tracks connections and filters packets in **chains**:

| Chain | Traffic | Example |
|---|---|---|
| `input` | **To** the router itself | WinBox, SSH, DNS queries to the router, ping to the router |
| `forward` | **Through** the router | LAN PC → internet |
| `output` | **From** the router itself | The router doing NTP or DNS lookups |

Rules are checked **top to bottom; the first match wins** (for terminating actions like accept and drop). If nothing matches, the packet is **accepted**. That's why every good firewall ends with drop rules.

### Connection states

| State | Meaning |
|---|---|
| `new` | First packet of a connection |
| `established` | Part of a known connection |
| `related` | Linked to a known connection (FTP data, ICMP errors) |
| `invalid` | Doesn't match any known state. Drop it. |
| `untracked` | Skipped tracking (set in `/ip firewall raw`) |

### Packet flow (simplified)

```
in → RAW prerouting → conntrack → mangle prerouting → dst-NAT → routing decision
   ├─ to router:  mangle input → FILTER INPUT → router process
   └─ through:    mangle forward → FILTER FORWARD → mangle postrouting → src-NAT → out
router process → mangle output → FILTER OUTPUT → src-NAT → out
```

### Actions

`accept`, `drop` (silent), `reject` (sends an ICMP or TCP reset), `add-src-to-address-list`, `add-dst-to-address-list`, `jump` / `return` (custom chains), `log`, `passthrough`, `fasttrack-connection`.

## 1. Interface lists (do this first)

Rules use **lists**, not interface names, so changing a WAN port means editing one place:

```
/interface list add name=WAN
/interface list add name=LAN
/interface list member add list=WAN interface=ether1
/interface list member add list=LAN interface=bridge1
```

## 2. Recommended base firewall (RouterOS 7 style)

This matches MikroTik's default-config logic. Use Safe Mode (`Ctrl+X`) while applying it.

### Input chain: protect the router

```
/ip firewall filter
add chain=input action=accept connection-state=established,related,untracked comment="IN: accept est/rel/untracked"
add chain=input action=drop connection-state=invalid comment="IN: drop invalid"
add chain=input action=accept protocol=icmp comment="IN: accept ICMP"
add chain=input action=accept dst-address=127.0.0.1 comment="IN: accept loopback (CAPsMAN)"
add chain=input action=drop in-interface-list=!LAN comment="IN: drop all not from LAN"
```

### Forward chain: protect the LAN

```
/ip firewall filter
add chain=forward action=fasttrack-connection connection-state=established,related hw-offload=yes comment="FWD: fasttrack"
add chain=forward action=accept connection-state=established,related,untracked comment="FWD: accept est/rel/untracked"
add chain=forward action=drop connection-state=invalid comment="FWD: drop invalid"
add chain=forward action=drop connection-state=new connection-nat-state=!dstnat in-interface-list=WAN comment="FWD: drop WAN not dstnat-ed"
```

What this gives you:
- LAN → internet: allowed.
- Internet → LAN: blocked, except port forwards (dst-nat).
- Internet → router: blocked, including WinBox, SSH and DNS.
- LAN → router: allowed.

> **Fasttrack** makes established traffic skip most of the firewall, mangle and queues. That means much faster routing, but **simple queues and mangle marks won't see fasttracked packets**. Disable it when doing QoS. See [K. QoS](../K-qos/).

## 3. Allow specific management from WAN

Put this **above** the "drop all not from LAN" rule (`place-before=`):

```
/ip firewall address-list add list=MGMT-ALLOWED address=203.0.113.50 comment="Office public IP"
/ip firewall filter add chain=input action=accept protocol=tcp dst-port=8291,22 src-address-list=MGMT-ALLOWED comment="IN: WinBox+SSH from trusted" place-before=[find comment="IN: drop all not from LAN"]
```

## 4. Address lists

```
/ip firewall address-list add list=BLOCKED address=198.51.100.0/24
/ip firewall address-list add list=BLOCKED address=example-bad.com      # resolved by DNS and updated automatically
/ip firewall address-list add list=TEMP address=1.2.3.4 timeout=1h       # auto-removed
/ip firewall address-list print where list=BLOCKED
```

Use them in rules with `src-address-list=` and `dst-address-list=`.

## 5. Brute-force protection (SSH example)

Staged lists. An IP that opens too many new connections climbs the stages and ends up blacklisted:

```
/ip firewall filter
add chain=input protocol=tcp dst-port=22 src-address-list=ssh_blacklist action=drop comment="SSH: drop blacklisted"
add chain=input protocol=tcp dst-port=22 connection-state=new src-address-list=ssh_stage3 action=add-src-to-address-list address-list=ssh_blacklist address-list-timeout=1d
add chain=input protocol=tcp dst-port=22 connection-state=new src-address-list=ssh_stage2 action=add-src-to-address-list address-list=ssh_stage3 address-list-timeout=1m
add chain=input protocol=tcp dst-port=22 connection-state=new src-address-list=ssh_stage1 action=add-src-to-address-list address-list=ssh_stage2 address-list-timeout=1m
add chain=input protocol=tcp dst-port=22 connection-state=new action=add-src-to-address-list address-list=ssh_stage1 address-list-timeout=1m
```

Place these above the input accept/drop rules that would otherwise handle SSH first.

## 6. Block LAN access to specific sites or IPs

```
/ip firewall address-list add list=BLOCK-SITES address=facebook.com
/ip firewall filter add chain=forward src-address=192.168.10.0/24 dst-address-list=BLOCK-SITES action=reject reject-with=icmp-network-unreachable place-before=0
```

For HTTPS sites behind CDNs, DNS-based address lists are imperfect. The **TLS SNI matcher** (`tls-host=*facebook.com`) is more precise, but it only works on the first packet with SNI and doesn't work with fasttrack.

## 7. RAW table (drop early, cheap)

The RAW table drops before connection tracking, which is good against floods:

```
/ip firewall raw add chain=prerouting src-address-list=BLOCKED action=drop
```

## 8. Logging

```
/ip firewall filter add chain=input action=log log-prefix="IN-DROP: " in-interface-list=WAN place-before=[find comment="IN: drop all not from LAN"]
/log print where message~"IN-DROP"
```

Remove log rules after debugging, since they're noisy.

## 9. IPv6 firewall

IPv6 has its **own** firewall (`/ipv6 firewall filter`). If IPv6 is enabled and you haven't configured it, you may be exposed. Either configure it the same way or disable IPv6:

```
/ipv6 settings set disable-ipv6=yes
```

## EVE-NG lab

```
[Cloud0 = "Internet"] --ether1(WAN)-- R1 --ether2 (bridge1, LAN)-- PC1
```

1. Set up interface lists, then apply the base firewall (section 2) using Safe Mode.
2. From your Windows PC (on the "WAN" side), try WinBox to `192.168.1.133`. It's **blocked**.
3. Add your PC's IP (`192.168.1.108`) to `MGMT-ALLOWED` and add the rule from section 3. WinBox **works**.
4. From PC1, ping 8.8.8.8 (once NAT from section F is done).
5. Add `8.8.8.8` to `BLOCK-SITES`. The ping from PC1 fails.
6. Watch the counters: `/ip firewall filter print stats`.

## Verify

```
/ip firewall filter print
/ip firewall filter print stats              # packet/byte counters per rule
/ip firewall connection print                # live connection table
/ip firewall address-list print
/tool torch interface=ether1                 # live traffic
```

## Troubleshooting

| Problem | Fix |
|---|---|
| Locked out | You applied the drop before the accept for your access. Use Safe Mode, console or MAC WinBox. |
| Rule never matches (0 counters) | A rule above it matches first, or the wrong chain (input vs forward). |
| Port forward doesn't work | You need the dst-nat rule. The base forward rule allows `connection-nat-state=dstnat`. |
| Queues or mangle ignored | Fasttrack. Disable the fasttrack rule. |
| Clients on the WAN side can use your DNS | `allow-remote-requests=yes` plus no input drop from WAN. Apply the base input chain. |
