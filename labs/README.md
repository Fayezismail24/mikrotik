# Labs: Hands-On EVE-NG Exercises

> **Status:** 🔄 Lab 1 done, Lab 2 in progress. More labs are added as the course goes.

Each lab is a complete exercise built in EVE-NG with MikroTik CHR, and each one has its own folder:

```
labs/lab-NN-name/
├── README.md       the lab (GitHub shows it when you open the folder)
├── qa.md           follow-up Q&A
├── topology.svg    the diagram
└── R1.rsc          router export, when added
```

Every lab README follows the same pattern:

**Goal → Topology → Concept → EVE-NG setup → Config steps (🖱️ WinBox + ⌨️ CLI) → Verify → Troubleshooting → Quick check**

- **Topology** (`topology.svg`) is drawn in layers: upstream (home router, Cloud0) at the top, the router in the middle, and the LAN/bridge box with its clients at the bottom. Interfaces and IPs are labelled on each link. A text version sits in a collapsible block underneath.
- **Troubleshooting** tables record the real mistakes made while doing the lab.
- **Quick check** is a short self-test.
- **Follow-up Q&A** (`qa.md`) goes deeper ("what if / why") and comes from reviewing the lab afterwards. Each answer is in a click-to-open box.

## Labs

| # | Lab | Topics | Cert | Status |
|---|---|---|---|---|
| 1 | [DHCP server, bridge and NAT](lab-01-dhcp-bridge-nat/) · [Q&A (33)](lab-01-dhcp-bridge-nat/qa.md) | WAN DHCP client, bridge with 4 LAN ports, DHCP server (pool + network), NAT masquerade, VPCS clients | MTCNA | ✅ Tested |
| 2 | [VLANs](lab-02-vlans/) | Fresh router, 6 VPCs on ether2–ether7, VLANs on the bridge | MTCNA · MTCRE | 🔄 In progress |

---

[← Back to the main index](../README.md)
