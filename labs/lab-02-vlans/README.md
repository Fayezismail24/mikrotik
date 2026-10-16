# Lab 2: VLANs

> **Cert:** MTCNA module 3 (Bridging) · MTCRE (VLANs) · **Status:** 🔄 In progress

[Labs index](../README.md) · [Back to the main index](../../README.md)

---

## Goal

One fresh CHR router that splits its LAN ports into separate VLANs, so groups of PCs are cut off from each other at Layer 2. This section is filled in as the lab goes.

## Topology

<details open><summary>Text version</summary>

```
Home router (192.168.1.1)
        │
   Cloud0 / pnet0  (home LAN 192.168.1.0/24)
        │
     ether1
     Mikrotik (CHR)
      ├─ ether2 ── VPC2
      ├─ ether3 ── VPC3
      ├─ ether4 ── VPC4
      ├─ ether5 ── VPC5
      ├─ ether6 ── VPC7
      └─ ether7 ── VPC6
```

</details>

## EVE-NG setup

A new lab file, **`Lab 2 - VLANs`**, with a fresh router. Lab 1 is not touched.

| Item | Value |
|---|---|
| Router | Node `Mikrotik`, MikroTik CHR 7.24.5 |
| Upstream | `Net` = Management (Cloud0), home LAN `192.168.1.0/24` |
| Clients | 6 × VPCS, cabled straight to the router |

### Port map

| Router port | Connected to | VLAN |
|---|---|---|
| ether1 | Net (Cloud0) | (uplink, no VLAN) |
| ether2 | VPC2 | *(set in Part 3)* |
| ether3 | VPC3 | *(set in Part 3)* |
| ether4 | VPC4 | *(set in Part 3)* |
| ether5 | VPC5 | *(set in Part 3)* |
| ether6 | VPC7 | *(set in Part 3)* |
| ether7 | VPC6 | *(set in Part 3)* |

> ⚠️ ether6 goes to **VPC7** and ether7 to **VPC6** (crossed on the canvas). Keep this in mind when you assign VLANs.
