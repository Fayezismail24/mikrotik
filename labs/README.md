# Labs: Hands-On EVE-NG Exercises

> **Status:** 🔄 Lab 1 done. More labs are added as the course goes.

Each lab is a complete exercise built in EVE-NG with MikroTik CHR. Every lab follows the same pattern:

**Goal → Topology → Concept → EVE-NG setup → Config steps (🖱️ WinBox + ⌨️ CLI) → Verify → Troubleshooting → Quick check → Follow-up questions**

- **Topology** is a diagram in `img/lab-NN-topology.svg`, drawn in layers: upstream (home router, Cloud0) at the top, the router in the middle, and the LAN/bridge box with its clients at the bottom. Interfaces and IPs are labelled on each link. A text version sits in a collapsible block underneath.
- **Troubleshooting** tables record the real mistakes made while doing the lab.
- **Quick check** is a short self-test. **Follow-up questions** go deeper ("what if / why") and come from reviewing the lab afterwards.

## Labs

| # | Lab | Topics | Cert | Status |
|---|---|---|---|---|
| 1 | [DHCP server, bridge and NAT](lab-01-dhcp-bridge-nat.md) | WAN DHCP client, bridge with 4 LAN ports, DHCP server (pool + network), NAT masquerade, VPCS clients | MTCNA | ✅ Tested |

---

[← Back to the main index](../README.md)
