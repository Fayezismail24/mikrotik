# 0. Fundamentals: Start Here

> **Status:** 🔄 Lesson 1 done. More lessons are added as the course goes.

This section starts from zero. Every lesson follows the same pattern:

**What it is → Why it exists → How it differs from similar things → Hands-on (🖱️ WinBox + ⌨️ CLI) → Quick check**

---

# Lesson 1: What is MikroTik, and what does a router do?

## 1.1 What is a router?

Your home has many devices (PC, phone, TV), and the internet is a huge outside network. A **router** sits **between networks** and decides where each packet goes.

**Analogy: a post office.**
- Every device has an **address** (its IP address), like a house address.
- When your PC sends something to Google, it hands it to the router, its "local post office".
- The router reads the destination address and decides which door (**interface**) to send it out of.

### What a router does

| Job | Plain meaning | Course section |
|---|---|---|
| **Routing** | Choosing the path to a destination | [G](../G-static-routing/), [H](../H-dynamic-routing/) |
| **Addressing (DHCP)** | Handing out addresses automatically | [C](../C-addressing/) |
| **NAT** | Many private devices sharing one public internet address | [F](../F-nat/) |
| **Firewall** | Deciding what's allowed in and out | [E](../E-firewall/) |
| **Switching / bridging** | Connecting devices inside one local network | [D](../D-bridging-vlans/) |

### Router vs switch vs access point

| Device | Works with | Job | Analogy |
|---|---|---|---|
| **Switch** | MAC addresses (Layer 2) | Connects devices **inside** one network | Hallways inside one building |
| **Router** | IP addresses (Layer 3) | Connects **different** networks | The post office between cities |
| **Access point** | Wireless | Lets Wi-Fi devices join a network | A door without a wall |

A home "router" is really all three in one box. A MikroTik can be all three as well.

## 1.2 What is MikroTik?

**MikroTik** is a Latvian company that makes networking gear. It's popular with ISPs and IT people because it's affordable and very powerful.

| Term | What it is |
|---|---|
| **RouterBOARD** | MikroTik **hardware** (hAP, CCR, CRS, cAP...) |
| **RouterOS** | The **operating system** that runs on every MikroTik device |
| **CHR** (Cloud Hosted Router) | RouterOS for **virtual machines**. This is R1 in our EVE-NG lab. |
| **WinBox** | The desktop GUI app for managing RouterOS |

**Why this matters:** what you learn on CHR works on any MikroTik, because it's the same RouterOS. The main difference is that CHR has no Wi-Fi hardware.

### RouterOS v6 vs v7

| | v6 | v7 |
|---|---|---|
| Status | Old, legacy | Current (this course uses 7.24.5) |
| Routing (OSPF/BGP) | Old menus | Fully rewritten, with different commands |
| New features | None | WireGuard, new Wi-Fi driver, REST API, containers |

If a command from a tutorial doesn't work, check whether the tutorial is for v6.

## 1.3 Interfaces: the router's doors

An **interface** is any connection point on the router.

| Kind | Examples | Notes |
|---|---|---|
| **Physical** | `ether1`, `ether2`, `sfp1`, `wifi1` | Real ports or radios |
| **Virtual** | `bridge1`, `vlan10`, `wg1`, `pppoe-out1` | Software-made doors, built later in the course |

In the lab, R1 has `ether1`–`ether4`. Only `ether1` is connected (to Cloud0, the home network).

## 1.4 Hands-on

### Look at R1's interfaces

🖱️ **WinBox:** **Interfaces** (left menu)

⌨️ **CLI** (WinBox → **New Terminal**):
```
/interface print
```

What to notice:
- **R** = **Running**: the link is up (cable "plugged in").
- **Type** `ether` = a physical Ethernet port.
- **MAC address**: every interface has its own hardware ID.
- **Tx / Rx** (WinBox) = live traffic out and in. ether1 moves; the others stay at 0.

### See what RouterOS this is

🖱️ **WinBox:** System → **Resources**

⌨️ **CLI:**
```
/system resource print
```

What to notice:
- **version**: `7.24.5`
- **board-name**: `CHR`, so it's virtual
- **architecture**: `x86_64`, running on a PC CPU

## ✅ Quick check

1. What's the difference between a router and a switch?
2. What's the difference between RouterOS, RouterBOARD and CHR?
3. In `/interface print`, which interfaces have the **R** flag, and why only those?

---

**Next:** Lesson 2: IP addresses and subnets.
