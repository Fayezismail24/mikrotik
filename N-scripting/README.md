# N. Scripting and Scheduler

> **Status:** 📝 Not yet tested in lab

## Concept

RouterOS scripting automates backups, failover, alerts and dynamic firewall lists.

- **Scripts:** 🖱️ System → **Scripts** (tabs: Scripts, Environment = global variables, Jobs = running). ⌨️ `/system script`
- **Scheduler:** 🖱️ System → **Scheduler**. ⌨️ `/system scheduler`
- **Event scripts** run from Netwatch up/down, DHCP leases, and PPP up/down.

Scripts are **written** in the RouterOS language either way. WinBox gives you an editor window; the CLI uses `source=`.

---

## 1. Language basics

Try these in **New Terminal**.

### Variables

```
:local name "R1"
:global counter 0
:set counter ($counter + 1)
:put $name
:log info "Hello from $name"
```

`:local` only exists inside its script or block. `:global` persists until reboot (see them in System → Scripts → **Environment**).

### Data types

| Type | Example |
|---|---|
| string | `"text"` |
| num | `42` |
| bool | `true` / `false` |
| ip / ip-prefix | `192.168.1.1`, `10.0.0.0/8` |
| time | `1h30m` |
| array | `{"a";"b"}` or `{name="R1"; ip=1.1.1.1}` |

### Conditions and loops

```
:if ($counter > 5) do={ :put "big" } else={ :put "small" }

:for i from=1 to=5 do={ :put $i }

:foreach iface in=[/interface find where running] do={
    :put [/interface get $iface name]
}

:while ($counter < 3) do={ :set counter ($counter + 1) }
```

### Reading config values

```
:local ip [/ip address get [find interface=ether1] address]
:local uptime [/system resource get uptime]
:put "ether1=$ip uptime=$uptime"
```

### Error handling

```
:do {
    /tool fetch url="https://example.com/file.txt" dst-path=file.txt
} on-error={ :log error "Download failed" }
```

### Built-ins

```
:put [:len "hello"]          # 5
:put [:pick "hello" 0 2]     # he
:put [:find "hello" "l"]     # 2
:put [:typeof 1.1.1.1]       # ip
:put [:resolve "google.com"]
:delay 5s
:put [/system clock get date]   # 2026-10-14 (v7.10+)
```

---

## 2. Creating and running scripts

🖱️ **WinBox:** System → **Scripts** → **+**:
- Name: `hello`
- Policy: tick what it needs (read, write, test...)
- Source: type or paste the script into the big box

Click **OK**, then select the script and click **Run Script**.

⌨️ **CLI:**
```
/system script add name=hello source=":log info \"hello world\""
/system script run hello
/system script print
```

> **Tip:** for long scripts, the WinBox editor is much easier. In the CLI you must escape `"` as `\"` and `$` as `\$` inside `source="..."`, or use the `source={ ... }` block form.

### Policies

A script can only do what its **Policy** boxes allow. The scheduler entry that runs it must have **at least** the same policies.

---

## 3. Scheduler

🖱️ **WinBox:** System → **Scheduler** → **+**:
- Name: `daily-backup`
- Start Time: `03:00:00` (or `startup` to run at boot)
- Interval: `1d 00:00:00`
- On Event: the script name (e.g. `auto-backup`) or inline code
- Policy: the same boxes as the script

⌨️ **CLI:**
```
/system scheduler add name=daily-backup interval=1d start-time=03:00:00 on-event=auto-backup policy=ftp,read,write,policy,test,sensitive
```

---

## 4. Practical scripts

For each one: 🖱️ System → Scripts → **+**, paste the code between `{ }` into **Source**, and tick the listed policies. ⌨️ Or paste the whole CLI block into New Terminal.

### Daily backup with a date

Policies: ftp, read, write, policy, test, sensitive.

```
/system script add name=auto-backup policy=ftp,read,write,policy,test,sensitive source={
    :local d [/system clock get date]
    :local id [/system identity get name]
    :local fname ("$id-$d")
    /system backup save name=$fname password="BackupPass"
    /export file=$fname show-sensitive
    :log info "Backup done: $fname"
}
```

### Email the backup

Add this inside the script after the export (Tools → Email must be configured, see [M](../M-monitoring/)):

```
:delay 5s
/tool e-mail send to="you@gmail.com" subject="Backup $id $d" file=("$fname.backup","$fname.rsc")
```

### Keep only the last 7 backups

Policies: read, write, ftp.

```
/system script add name=cleanup-backups policy=read,write,ftp source={
    :local files [/file find where name~"\\.backup\$"]
    :local count [:len $files]
    :if ($count > 7) do={
        :for i from=0 to=($count - 8) do={
            /file remove ($files->$i)
        }
    }
}
```

### Track the current WAN IP in an address list (for hairpin NAT)

Policies: read, write.

```
/system script add name=update-wan-ip policy=read,write source={
    :local wan [/ip address get [find interface=ether1] address]
    :set wan [:pick $wan 0 [:find $wan "/"]]
    /ip firewall address-list remove [find list=WAN-IP]
    /ip firewall address-list add list=WAN-IP address=$wan
}
/system scheduler add name=wan-ip interval=5m on-event=update-wan-ip policy=read,write
```

### Run at boot

🖱️ **WinBox:** Scheduler → **+** → Start Time `startup`, On Event `:delay 30s; :log info "Router booted"`

⌨️ **CLI:**
```
/system scheduler add name=on-boot start-time=startup on-event=":delay 30s; :log info \"Router booted\""
```

### DHCP lease script (log new devices)

🖱️ **WinBox:** IP → DHCP Server → **DHCP** tab → double-click `dhcp-lan` → **Script** tab → paste the code below.

⌨️ **CLI:**
```
/ip dhcp-server set dhcp-lan lease-script={
    :if ($leaseBound = 1) do={
        :log info "New lease: $leaseActIP $leaseActMAC"
    }
}
```

---

## 5. Import .rsc files

🖱️ **WinBox:** drag the `.rsc` into **Files**, then in **New Terminal** run the import command below. There's no import button.

⌨️ **CLI:**
```
/import file-name=my-setup.rsc
/import file-name=my-setup.rsc verbose=yes
```

---

## 6. REST API (v7)

🖱️ **WinBox:** IP → **Services** → enable `www-ssl` with a certificate (or `www` for lab only)

⌨️ **CLI:**
```
/ip service enable www
```

From Windows PowerShell:

```powershell
curl.exe -k -u admin:password https://192.168.1.133/rest/system/resource
curl.exe -k -u admin:password https://192.168.1.133/rest/ip/address
curl.exe -k -u admin:password -X PUT https://192.168.1.133/rest/ip/address -H "content-type: application/json" -d '{\"address\":\"10.9.9.1/24\",\"interface\":\"ether3\"}'
```

Python alternatives: the `librouteros` or `routeros_api` libraries (API service, port 8728/8729).

---

## EVE-NG lab

1. Create `auto-backup` in WinBox and schedule it every 5 minutes. Watch **Files** fill up.
2. Add and run `cleanup-backups`. Only 7 remain.
3. In New Terminal, run a `:foreach` that prints every interface name.
4. Add the DHCP lease script, connect a VPCS with `ip dhcp`, and watch the **Log**.
5. Query R1 over REST from PowerShell.

## Verify

| Check | 🖱️ WinBox | ⌨️ CLI |
|---|---|---|
| Scripts + run count | System → Scripts | `/system script print` |
| Running jobs | System → Scripts → **Jobs** | `/system script job print` |
| Global variables | System → Scripts → **Environment** | `/system script environment print` |
| Schedules + next run | System → Scheduler | `/system scheduler print` |

## Troubleshooting

| Problem | Fix |
|---|---|
| Runs manually, not from the scheduler | Scheduler Policy missing permissions. Match the script's. |
| `syntax error` in the CLI | Unescaped `"` or `$`. Use the WinBox editor or `source={ }`. |
| Global variable empty | Globals reset on reboot. Declare `:global name` before reading it. |
| Bad filename from the date | v7.10+ is `YYYY-MM-DD`. Older versions use `oct/14/2026` (the `/` is invalid). |
| Script silently does nothing | Add `:log info` lines and check the **Log**. |
