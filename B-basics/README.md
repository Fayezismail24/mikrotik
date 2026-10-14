# B. Basics: CLI, WinBox, Users, Backups, Upgrades

> **Status:** 🔄 In progress. Identity, clock/NTP and users tested on CHR 7.24.5 with WinBox.

## Concept

RouterOS is managed in four main ways:

| Tool | What it is | Use it for |
|---|---|---|
| **WinBox** 🖱️ | Desktop GUI app, connects by IP or MAC | Visual config, monitoring |
| **CLI** ⌨️ | Terminal over console, SSH or WinBox **New Terminal** | Fast config, scripting, copy/paste |
| **WebFig** | Web UI at `http://router-ip` | When WinBox isn't available |
| **REST API** | `https://router-ip/rest/...` (v7) | Automation |

Every WinBox action has a matching CLI command. This guide shows **both** for every step.

---

## 1. WinBox: connecting

🖱️ **WinBox:**
1. Download WinBox from mikrotik.com/download. It's a single file, no install.
2. **Neighbors** tab: lists routers on the same L2 network.
3. Connect by **IP** (type the IP, user, password) or by **MAC** (click the MAC in Neighbors). MAC works even with no IP configured, which is great for recovery.
4. **Add/Set** saves the session for next time.

Useful WinBox habits:
- **New Terminal** in the left menu opens the CLI inside WinBox.
- Double-click any item to edit it.
- Right-click a table header → **Show Columns** to add fields.
- Use the **Comment** button (yellow note) to label items.
- **Safe Mode** is the button at the top-left of the window.

⌨️ **CLI (SSH from Windows):**
```bash
ssh admin@192.168.1.133
```

---

## 2. CLI essentials

There's no WinBox equivalent; this is how the terminal works.

| Key / command | Action |
|---|---|
| `Tab` | Auto-complete. Press twice to list the options. |
| `?` | Help for the current position |
| `↑` / `↓` | Command history |
| `Ctrl+X` | Toggle **Safe Mode** |
| `Ctrl+C` | Stop a running command |
| `/ip address` | Enter a menu |
| `..` / `/` | Up one level / root |
| `print`, `print detail`, `print terse` | Show items |
| `print where interface=ether1` | Filter |
| `set [find interface=ether1] comment="WAN"` | Edit by query instead of number |

---

## 3. System info ✅

🖱️ **WinBox:**
- **System → Resources**: version, uptime, CPU, free memory, architecture.
- **System → Identity**: the router name.
- **Interfaces**: all ports, with live traffic.
- **IP → Addresses**: IPs per interface.

⌨️ **CLI:**
```
/system resource print
/system identity print
/interface print
/ip address print
```

---

## 4. Identity ✅

🖱️ **WinBox:** System → Identity → Identity: `R1` → **OK**

⌨️ **CLI:**
```
/system identity set name=R1
```

---

## 5. Clock and NTP ✅

🖱️ **WinBox:**
1. **System → Clock** → Time Zone Name: pick yours (e.g. `Asia/Beirut`) → **Apply**.
2. **System → NTP Client**:
   - Tick **Enabled**.
   - Servers: `pool.ntp.org`.
   - Click **Apply**.
3. Wait about 10 seconds. The **Status** field shows `synchronized`.
4. Back in **System → Clock**, confirm the time and date are correct.

⌨️ **CLI:**
```
/system clock set time-zone-name=Asia/Beirut
/system ntp client set enabled=yes
/system ntp client servers add address=pool.ntp.org
/system ntp client print
/system clock print
```

---

## 6. Users and groups ✅

Default groups: `full` (everything), `write` (config, no user management), `read` (view only).

> **Before you start:** keep the EVE-NG console to the router open. If you lock yourself out in WinBox, you can still log in there.

### 6.1 Create your own admin user

🖱️ **WinBox:** System → Users → **Users** tab → **+**:
- Name: `yourname`
- Group: `full`
- Password: a strong one, typed twice

Then click **OK**.

⌨️ **CLI:**
```
/user add name=yourname group=full password="StrongPass!"
```

### 6.2 Test the new user before touching admin

🖱️ **WinBox:** **File → New Window**, then connect to the router with the new user. Only continue if this login works.

⌨️ **CLI:**
```bash
ssh yourname@192.168.1.133
```

### 6.3 Disable the default admin

🖱️ **WinBox:** while logged in as your new user, go to System → Users → select `admin` → click the red **✕** (Disable). The row turns grey.

⌨️ **CLI:**
```
/user disable admin
```

### 6.4 See who is logged in

🖱️ **WinBox:** System → Users → **Active Users** tab

⌨️ **CLI:**
```
/user active print
```

### 6.5 Custom group (example: read-only NOC user)

🖱️ **WinBox:** System → Users → **Groups** tab → **+**:
- Name: `noc`
- Tick: `read`, `test`, `winbox`, `ssh`
- Leave unticked: `write`, `policy`

Click **OK**, then go to the **Users** tab and add a user with group `noc`.

⌨️ **CLI:**
```
/user group add name=noc policy=read,test,winbox,ssh,!write,!policy
/user add name=noc1 group=noc password="NocPass!"
```

### 6.6 SSH key login

🖱️ **WinBox:**
1. Open **Files** and drag `id_ed25519.pub` from your PC into the window.
2. Go to System → Users → **SSH Keys** tab → **Import SSH Key**.
3. Choose the user and the key file.

⌨️ **CLI:**
```
/user ssh-keys import public-key-file=id_ed25519.pub user=yourname
```

---

## 7. Backups

| Type | Format | Notes |
|---|---|---|
| **Binary backup** | `.backup` | Full restore on the **same** device or model. Includes users and passwords. Not readable. |
| **Export** | `.rsc` text | Readable and portable. Passwords are hidden unless you add `show-sensitive`. |

### 7.1 Binary backup

🖱️ **WinBox:** **Files** → **Backup** button:
- Name: `R1-2026-10-14`
- Password: optional, recommended

Click **Backup**. The file appears in the Files list.

⌨️ **CLI:**
```
/system backup save name=R1-2026-10-14 password="BackupPass"
```

### 7.2 Export (text config)

🖱️ **WinBox:** there's no menu button for this. Open **New Terminal** and use the CLI command below. The file then appears in **Files**.

⌨️ **CLI:**
```
/export file=R1-config
/export file=R1-config-full show-sensitive
/export compact                       # print to screen
/ip firewall export                   # one menu only
```

### 7.3 Download backups to your PC

🖱️ **WinBox:** **Files** → drag the file onto your desktop.

⌨️ **CLI (from Windows):**
```bash
scp yourname@192.168.1.133:R1-config.rsc .
```

### 7.4 Restore

🖱️ **WinBox:**
- **Binary:** **Files** → select the `.backup` → **Restore** → enter the password. The router reboots.
- **Export:** drag the `.rsc` into **Files**, then in **New Terminal** run `/import file-name=R1-config.rsc`.

⌨️ **CLI:**
```
/system backup load name=R1-2026-10-14.backup password="BackupPass"
/import file-name=R1-config.rsc
```

**Rule of thumb:** keep both. `.backup` is for a fast same-box restore, `.rsc` is for documentation and migration.

---

## 8. Upgrades

🖱️ **WinBox:** System → Packages → **Check For Updates**:
- Channel: `stable`
- Click **Check For Updates**.
- If a new version is listed, click **Download&Install**. The router reboots.

⌨️ **CLI:**
```
/system package update set channel=stable
/system package update check-for-updates
/system package update install
```

Channels: `long-term` (most conservative), `stable`, `testing`, `development`.

### Firmware (physical RouterBOARDs only, not CHR)

🖱️ **WinBox:** System → RouterBOARD → **Upgrade**, then System → **Reboot**

⌨️ **CLI:**
```
/system routerboard upgrade
/system reboot
```

### Packages

🖱️ **WinBox:** System → Packages → select a package → **Disable** / **Enable**. Changes apply after a reboot.

⌨️ **CLI:**
```
/system package print
/system package disable wireless
```

---

## 9. Safe Mode

If your session drops while Safe Mode is on, RouterOS **undoes every change** made since you turned it on.

🖱️ **WinBox:** click **Safe Mode** (top-left) → make changes → click **Safe Mode** again to keep them.

⌨️ **CLI:** press `Ctrl+X` → make changes → press `Ctrl+X` again to keep them.

---

## 10. Reset and recovery

🖱️ **WinBox:** System → Reset Configuration:
- **No Default Configuration** = completely empty.
- **Do Not Backup** = skip the automatic backup before reset.

Click **Reset Configuration**.

⌨️ **CLI:**
```
/system reset-configuration
/system reset-configuration no-defaults=yes skip-backup=yes
```

Other recovery options:
- **Reset button** on physical devices: hold it while powering on.
- **Netinstall:** reinstalls RouterOS over the network. It's the last resort.
- **Lost password:** there's no recovery; reset or Netinstall.

---

## 11. Reboot and shutdown

🖱️ **WinBox:** System → **Reboot** / **Shutdown**

⌨️ **CLI:**
```
/system reboot
/system shutdown
```

In EVE-NG, prefer Shutdown over **Stop** to avoid the "rebooted without proper shutdown" log message.

---

## EVE-NG lab

**Topology:** R1 ether1 → Cloud0.

1. Set identity, time zone and NTP.
2. Create your own `full` user, test it in a second WinBox window, then disable `admin`.
3. Take a `.backup` and an `/export file=`, and download both to your PC.
4. Change the identity to `TEST`, restore the backup, and confirm it's back to `R1`.
5. Safe Mode test: turn on Safe Mode, remove the IP from ether1, and close WinBox. Reconnect after about 10 seconds; the IP should be back.

## Lab progress

- ✅ System info
- ✅ Identity
- ✅ Clock / NTP
- ✅ Users (own full user, admin disabled)
- ⬜ Backups (binary + export + restore)
- ⬜ Upgrades
- ⬜ Safe Mode
- ⬜ Reset

## Troubleshooting

| Problem | Fix |
|---|---|
| WinBox Neighbors shows nothing | Different L2 network, or MAC server / discovery restricted. Connect by IP. |
| Locked out after a change | Use Safe Mode next time. Recover with MAC WinBox or the console. |
| `.backup` restore on another model misbehaves | Use an `.rsc` export instead and adjust interface names. |
| Export missing passwords | Add `show-sensitive`. |
| NTP not syncing | Check DNS (`/ping pool.ntp.org`) and that UDP 123 isn't blocked. |
