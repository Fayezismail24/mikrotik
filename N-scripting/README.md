# N. Scripting and Scheduler

> **Status:** 📝 Not yet tested in lab

## Concept

RouterOS has its own scripting language for automating tasks: backups, failover, alerts, dynamic firewall lists, and more.

- **Scripts** are stored in `/system script`.
- **Scheduler** runs scripts on a timer or at startup (`/system scheduler`).
- **Event scripts** run when something changes: Netwatch up/down, DHCP lease, PPP up/down.

## 1. Language basics

### Variables

```
:local name "R1"          # only exists inside this script or block
:global counter 0         # persists until reboot, visible to other scripts
:set counter ($counter + 1)
:put $name                # print to terminal
:log info "Hello from $name"
```

### Data types

| Type | Example |
|---|---|
| string | `"text"` |
| num | `42` |
| bool | `true` / `false` |
| ip / ip-prefix | `192.168.1.1`, `10.0.0.0/8` |
| time | `1h30m`, `00:10:00` |
| array | `{"a";"b";"c"}` or `{name="R1"; ip=1.1.1.1}` |

### Conditions

```
:if ($counter > 5) do={
    :put "big"
} else={
    :put "small"
}
```

### Loops

```
:for i from=1 to=5 do={ :put $i }

:foreach iface in=[/interface find where running] do={
    :put [/interface get $iface name]
}

:while ($counter < 3) do={ :set counter ($counter + 1) }
```

### Getting values from config

```
:local ip [/ip address get [find interface=ether1] address]
:local uptime [/system resource get uptime]
:local ver [/system resource get version]
:put "ether1=$ip uptime=$uptime version=$ver"
```

### Error handling

```
:do {
    /tool fetch url="https://example.com/file.txt" dst-path=file.txt
} on-error={
    :log error "Download failed"
}
```

### Useful built-ins

```
:put [:len "hello"]                     # 5
:put [:pick "hello" 0 2]                # "he"
:put [:find "hello" "l"]                # 2
:put [:typeof 1.1.1.1]                  # ip
:put [:resolve "google.com"]
:delay 5s
:put [/system clock get date]           # 2026-10-14 (v7.10+ format)
```

## 2. Script management

```
/system script add name=hello source=":log info \"hello world\""
/system script run hello
/system script print
/system script remove hello
```

For long scripts, edit them in WinBox (System → Scripts) or paste them in a multi-line block. Quotes and `$` inside `source=` must be escaped (`\"`, `\$`) when adding from the CLI.

### Policies

Each script has `policy=` (read, write, policy, test, ftp, reboot, sensitive...). A script can only do what its policies allow. The scheduler's policies must include the script's policies.

## 3. Practical scripts

### Daily backup with a date in the filename

```
/system script add name=auto-backup policy=ftp,read,write,policy,test,sensitive source={
    :local d [/system clock get date]
    :local id [/system identity get name]
    :local fname ("$id-$d")
    /system backup save name=$fname password="BackupPass"
    /export file=$fname show-sensitive
    :log info "Backup done: $fname"
}
/system scheduler add name=daily-backup interval=1d start-time=03:00:00 on-event=auto-backup policy=ftp,read,write,policy,test,sensitive
```

### Email the backup

Add this inside the script after the export (needs `/tool e-mail` configured, see [M. Monitoring](../M-monitoring/)):

```
:delay 5s
/tool e-mail send to="you@gmail.com" subject="Backup $id $d" file=("$fname.backup","$fname.rsc")
```

### Delete old backups (keep the last 7)

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

`/file find` returns items ordered by name, which here means ordered by date because of the filename format.

### Update an address list with the current WAN IP (for hairpin NAT)

```
/system script add name=update-wan-ip source={
    :local wan [/ip address get [find interface=ether1] address]
    :set wan [:pick $wan 0 [:find $wan "/"]]
    /ip firewall address-list remove [find list=WAN-IP]
    /ip firewall address-list add list=WAN-IP address=$wan
}
/system scheduler add name=wan-ip interval=5m on-event=update-wan-ip
```

### Run at boot

```
/system scheduler add name=on-boot start-time=startup on-event=":delay 30s; :log info \"Router booted\""
```

### DHCP lease script (log new devices)

```
/ip dhcp-server set dhcp-lan lease-script={
    :if ($leaseBound = 1) do={
        :log info "New lease: $leaseActIP $leaseActMAC $"lease-hostname""
    }
}
```

## 4. Import and run .rsc files

Any `.rsc` file is a script of commands:

```
/import file-name=my-setup.rsc
/import file-name=my-setup.rsc verbose=yes     # shows which line fails
```

## 5. REST API (v7, automation from a PC)

Enable HTTPS (`www-ssl`) with a certificate, then from Windows PowerShell:

```powershell
curl.exe -k -u admin:password https://192.168.1.133/rest/system/resource
curl.exe -k -u admin:password https://192.168.1.133/rest/ip/address
curl.exe -k -u admin:password -X PUT https://192.168.1.133/rest/ip/address -H "content-type: application/json" -d '{\"address\":\"10.9.9.1/24\",\"interface\":\"ether3\"}'
```

Plain HTTP (`www`) also serves `/rest`, but only use that in the lab.

Python alternative: the `librouteros` or `routeros_api` libraries use the API service (port 8728/8729).

## EVE-NG lab

1. Write `auto-backup` and schedule it every 5 minutes for testing. Watch `/file print` fill up.
2. Write `cleanup-backups` and run it. Only 7 files remain.
3. Make a `:foreach` that prints every interface and its running status.
4. Add the DHCP lease script and connect a VPCS with `ip dhcp`. Watch the log.
5. Query R1 over REST from Windows PowerShell.

## Troubleshooting

| Problem | Fix |
|---|---|
| Script runs manually but not from the scheduler | The scheduler policy is missing permissions the script needs. Match them. |
| `syntax error` when adding from the CLI | Unescaped quotes or `$`. Use the `source={ ... }` block form, or edit in WinBox. |
| Global variable empty | Globals reset on reboot, and a script must declare `:global name` before reading it. |
| Date format breaks the filename | v7.10+ uses `YYYY-MM-DD`. Older versions use `oct/14/2026`, where the `/` is invalid in filenames. |
| Script silently does nothing | Add `:log info` lines to trace it, and run with `/system script run name` to see errors. |
