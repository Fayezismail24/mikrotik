# B. Basics: CLI, WinBox, Users, Backups, Upgrades

> **Status:** 📝 Not yet tested in lab

## Concept

RouterOS is managed in four main ways:

| Tool | What it is | Use it for |
|---|---|---|
| **CLI** | Terminal over console, SSH or WinBox "New Terminal" | Fast config, scripting, copy/paste |
| **WinBox** | Windows GUI app, connects by IP or MAC | Visual config, monitoring |
| **WebFig** | Web UI at `http://router-ip` | When WinBox isn't available |
| **REST API** | `https://router-ip/rest/...` (v7) | Automation |

Everything you do in WinBox has a matching CLI command. Learn both.

## 1. CLI essentials

### Navigation

```
/ip address          # go into a menu
print                # show items in the current menu
..                   # go up one level
/                    # go to the root
/ip address print    # run a command from anywhere using the full path
```

### Keys

| Key | Action |
|---|---|
| `Tab` | Auto-complete. Press twice to list the options. |
| `?` | Help for the current position |
| `↑` / `↓` | Command history |
| `Ctrl+X` | Toggle **Safe Mode** |
| `Ctrl+C` | Stop a running command (ping, monitor) |
| `F1` | Help |

### Print variations

```
/interface print
/interface print detail          # all properties
/interface print terse           # one line per item, good for copying
/interface print where running   # filter
/ip address print where interface=ether1
/interface monitor-traffic ether1   # live
```

### Editing items

Items are referenced by the number from the last `print`, or with `find`:

```
/ip address print
/ip address set 0 comment="WAN"
/ip address set [find interface=ether1] comment="WAN"
/ip address disable 0
/ip address remove [find address="10.0.0.1/24"]
```

### Safe Mode

Press `Ctrl+X` before risky changes. If your session drops (for example you lock yourself out), RouterOS **reverts every change made in Safe Mode**. Press `Ctrl+X` again to keep the changes.

## 2. WinBox

1. Download from mikrotik.com/download. It's a single file; WinBox 4 runs on Windows, macOS and Linux.
2. **Neighbors tab:** finds routers on the same L2 network via MNDP.
3. **Connect by MAC:** works even when the router has no IP. Click the MAC address in Neighbors. Great for recovery.
4. **Connect by IP:** type the IP, user and password.
5. Save sessions with **Add/Set**.

Useful WinBox features:
- **New Terminal**: the CLI inside WinBox.
- **Columns**: right-click a table header to add fields.
- Double-click any item to edit it.
- **Comment** button: label everything.

## 3. Identity, clock, NTP

```
/system identity set name=R1
/system clock set time-zone-name=Asia/Beirut
/system ntp client set enabled=yes
/system ntp client servers add address=pool.ntp.org
/system clock print
```

Use your own time zone name. Tab-complete it after `time-zone-name=`.

## 4. Users and groups

Default groups: `full` (everything), `write` (config, no user management), `read` (view only).

```
/user add name=fayez group=full password="StrongPass!"
/user print
/user group print
/user active print           # who is logged in now
```

**Best practice:** create your own full user, log in as it, then disable or remove `admin`:

```
/user disable admin
```

Custom group example (read-only plus ping):

```
/user group add name=noc policy=read,test,winbox,ssh,!write,!policy
/user add name=noc1 group=noc password="NocPass!"
```

SSH key login:

```
/file print                                    # upload id_ed25519.pub first (WinBox Files drag-drop)
/user ssh-keys import public-key-file=id_ed25519.pub user=fayez
```

## 5. Backups

| Type | Command | Format | Notes |
|---|---|---|---|
| **Binary backup** | `/system backup save` | `.backup` | Full restore on the **same** device or model. Includes users and passwords. Not human-readable. |
| **Export** | `/export` | `.rsc` text | Readable and portable to other devices. Passwords are hidden unless you add `show-sensitive`. |

```
/system backup save name=R1-2026-10-14 password="BackupPass"
/export file=R1-config
/export file=R1-config-full show-sensitive
/export compact                     # print to screen, only non-default settings
/ip firewall export                 # export just one menu
/file print
```

Restore:

```
/system backup load name=R1-2026-10-14.backup password="BackupPass"
/import file-name=R1-config.rsc
```

Download files off the router through WinBox **Files** (drag to desktop), or with `scp`:

```bash
scp admin@192.168.1.133:R1-config.rsc .
```

**Rule of thumb:** keep both. `.backup` is for a fast same-box restore, `.rsc` is for documentation and migration.

## 6. Upgrades

```
/system package update set channel=stable
/system package update check-for-updates
/system package update install          # downloads, then reboots
```

Channels: `long-term` (most conservative), `stable`, `testing`, `development`.

On physical RouterBOARDs, also upgrade the **firmware** after RouterOS:

```
/system routerboard print
/system routerboard upgrade
/system reboot
```

CHR has no RouterBOARD firmware.

Packages:

```
/system package print
/system package disable wireless     # takes effect after reboot
```

## 7. Reset and recovery

```
/system reset-configuration                         # back to default config
/system reset-configuration no-defaults=yes         # completely empty
/system reset-configuration no-defaults=yes skip-backup=yes
```

Other recovery options:
- **Physical reset button:** hold it while powering on. Release when the LED flashes for a default-config reset; keep holding for Netinstall mode.
- **Netinstall:** a Windows/Linux tool that reinstalls RouterOS over the network. It's the last resort for a bricked or locked-out device.
- **Lost password:** there's no recovery. Reset or Netinstall.

## 8. Reboot and shutdown

```
/system reboot
/system shutdown
```

In EVE-NG, prefer `/system shutdown` over **Stop** to avoid the "rebooted without proper shutdown" log message.

## EVE-NG lab

**Topology:** R1 ether1 → Cloud0.

1. Set identity `R1`, time zone, and NTP.
2. Create user `fayez` (full), log in with it over SSH from Windows: `ssh fayez@192.168.1.133`.
3. Disable `admin`.
4. Take a `.backup` and an `/export file=`, then download both with WinBox.
5. Change the identity to `TEST`, then restore the backup and confirm it's back to `R1`.
6. Practice Safe Mode: press `Ctrl+X`, remove the IP from ether1, and close the window. Reconnect after about 10 seconds; the IP should be back.

## Verify

```
/system identity print
/system clock print
/user print
/file print
/system resource print       # version, uptime, CPU, RAM
/system package update print
```

## Troubleshooting

| Problem | Fix |
|---|---|
| WinBox Neighbors shows nothing | Different L2 network, or MAC server / discovery restricted. Connect by IP. |
| Locked out after a change | Use Safe Mode next time. Recover with MAC WinBox or console. |
| `.backup` restore on a different model misbehaves | Use an `.rsc` export instead, and edit interface names to fit. |
| Export missing passwords | Add `show-sensitive`. |
| NTP not syncing | Check DNS works (`/ping pool.ntp.org`) and UDP 123 isn't blocked. |
