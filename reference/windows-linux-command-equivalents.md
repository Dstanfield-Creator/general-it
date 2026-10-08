# Windows and Linux Command Equivalents

> Side-by-side reference of Windows (cmd and PowerShell) and Linux commands for everyday administration tasks, plus notes on translating PowerShell habits to bash.

**Status:** Active · **Updated:** 2026-10-08

## How to read the tables

The Windows column lists the cmd form first where one exists, then the PowerShell form; cmdlets work in Windows PowerShell 5.1 and PowerShell 7 unless noted. The Linux column assumes a systemd-based distribution with GNU coreutils; package commands show Debian/Ubuntu (`apt`) then RHEL/Fedora (`dnf`). Hosts and addresses use `example.com` and 192.0.2.x.

## Files and directories

| Task | Windows | Linux |
|------|---------|-------|
| List directory contents | `dir`; `Get-ChildItem` | `ls -la` |
| Create directory with parents | `mkdir a\b\c`; `New-Item -ItemType Directory -Force a\b\c` | `mkdir -p a/b/c` |
| Copy a file or tree | `copy src dst`; `robocopy src dst /E`; `Copy-Item src dst -Recurse` | `cp src dst`; `cp -a src dst`; `rsync -a src/ dst/` |
| Move or rename | `move src dst`; `Move-Item src dst` | `mv src dst` |
| Delete a tree | `rmdir /S /Q dir`; `Remove-Item dir -Recurse -Force` | `rm -rf dir` |
| Print a file, first or last lines | `type file`; `Get-Content file -Head 20`; `Get-Content file -Tail 20` | `cat file`; `head -n 20 file`; `tail -n 20 file` |
| Follow a growing file | `Get-Content file -Wait` | `tail -f file` |
| Find files by name | `dir /S /B *.log`; `Get-ChildItem -Recurse -Filter *.log` | `find . -name '*.log'` |
| Directory size | `Get-ChildItem dir -Recurse \| Measure-Object Length -Sum` | `du -sh dir` |
| File hash | `certutil -hashfile file SHA256`; `Get-FileHash file` | `sha256sum file` |
| Archive and extract | `tar -cf a.tar dir`; `Compress-Archive dir a.zip`; `Expand-Archive a.zip` | `tar -czf a.tgz dir`; `tar -xzf a.tgz`; `unzip a.zip` |

## Searching text

| Task | Windows | Linux |
|------|---------|-------|
| Search a file for a string | `findstr "text" file`; `Select-String "text" file` | `grep "text" file` |
| Search recursively, case-insensitive | `findstr /S /I "text" *.conf`; `Get-ChildItem -Recurse \| Select-String "text"` | `grep -ri "text" .` |
| Regex search | `findstr /R "^err.*" file`; `Select-String -Pattern "^err" file` | `grep -E "^err" file` |
| Count matches | `(Select-String "text" file).Count` | `grep -c "text" file` |
| Replace text in a file | `(Get-Content f) -replace "old","new" \| Set-Content f` | `sed -i 's/old/new/g' f` |
| Extract a column | `... \| ForEach-Object { ($_ -split '\s+')[2] }` | `awk '{print $3}'`; `cut -d' ' -f3` |
| Sort and de-duplicate | `Sort-Object -Unique` | `sort -u` |

## Processes and services

| Task | Windows | Linux |
|------|---------|-------|
| List processes | `tasklist`; `Get-Process` | `ps aux`; `top`; `htop` |
| Find a process by name | `tasklist /FI "IMAGENAME eq nginx.exe"`; `Get-Process nginx` | `pgrep -a nginx`; `ps aux \| grep nginx` |
| Kill a process | `taskkill /PID 1234 /F`; `Stop-Process -Id 1234 -Force` | `kill 1234`; `kill -9 1234`; `pkill nginx` |
| Service status | `sc query spooler`; `Get-Service spooler` | `systemctl status cups` |
| Start or stop a service | `net start spooler`; `Start-Service spooler`; `Stop-Service spooler` | `sudo systemctl start cups`; `sudo systemctl stop cups` |
| Enable at boot | `Set-Service spooler -StartupType Automatic` | `sudo systemctl enable cups` |
| List all services | `Get-Service` | `systemctl list-units --type=service` |
| Which process owns a port | `netstat -ano \| findstr :443`; `Get-NetTCPConnection -LocalPort 443` | `ss -tlnp 'sport = :443'`; `lsof -i :443` |

## Networking

| Task | Windows | Linux |
|------|---------|-------|
| Interface addresses | `ipconfig /all`; `Get-NetIPAddress` | `ip -br addr` |
| Link state | `Get-NetAdapter` | `ip -br link` |
| Routing table | `route print`; `Get-NetRoute` | `ip route` |
| Renew DHCP lease | `ipconfig /release && ipconfig /renew` | `sudo dhclient -v eth0`; `nmcli connection up eth0` |
| DNS lookup | `nslookup app.example.com`; `Resolve-DnsName app.example.com` | `dig app.example.com`; `resolvectl query app.example.com` |
| Flush DNS cache | `ipconfig /flushdns`; `Clear-DnsClientCache` | `resolvectl flush-caches` |
| Trace route | `tracert 192.0.2.80`; `Test-NetConnection 192.0.2.80 -TraceRoute` | `traceroute 192.0.2.80`; `mtr -rw 192.0.2.80` |
| Test a TCP port | `Test-NetConnection 192.0.2.80 -Port 443` | `nc -zv 192.0.2.80 443` |
| Listening sockets | `netstat -an \| findstr LISTEN`; `Get-NetTCPConnection -State Listen` | `ss -tlnp` |
| HTTP request | `Invoke-WebRequest https://app.example.com/`; `curl.exe -I https://app.example.com/` | `curl -I https://app.example.com/`; `wget -S --spider https://app.example.com/` |
| Firewall rules | `Get-NetFirewallRule`; `netsh advfirewall firewall show rule name=all` | `sudo nft list ruleset`; `sudo ufw status verbose` |

## Users and groups

| Task | Windows | Linux |
|------|---------|-------|
| Current user and groups | `whoami`; `whoami /groups` | `whoami`; `id` |
| List local users | `net user`; `Get-LocalUser` | `getent passwd`; `cat /etc/passwd` |
| Create a local user | `net user name * /add`; `New-LocalUser name` | `sudo useradd -m name`; `sudo adduser name` |
| Add to a group | `net localgroup Administrators name /add`; `Add-LocalGroupMember Administrators name` | `sudo usermod -aG sudo name` |
| Group members | `net localgroup Administrators`; `Get-LocalGroupMember Administrators` | `getent group sudo`; `groups name` |
| Run as another user | `runas /user:name cmd`; `Start-Process cmd -Verb RunAs` | `sudo -u name cmd`; `su - name` |

## Disks and storage

| Task | Windows | Linux |
|------|---------|-------|
| Disk free space | `wmic logicaldisk get size,freespace`; `Get-PSDrive -PSProvider FileSystem` | `df -h` |
| List disks and partitions | `diskpart` then `list disk`; `Get-Disk`; `Get-Partition` | `lsblk`; `sudo fdisk -l` |
| Mount a network share | `net use Z: \\fs.example.com\share`; `New-PSDrive -Name Z -PSProvider FileSystem -Root \\fs.example.com\share` | `sudo mount -t cifs //fs.example.com/share /mnt -o username=name`; `sudo mount 192.0.2.80:/export /mnt` |
| Check a filesystem | `chkdsk C: /F`; `Repair-Volume C -Scan` | `sudo fsck /dev/sdb1` (unmounted) |
| Largest files | `Get-ChildItem -Recurse \| Sort-Object Length -Descending \| Select-Object -First 10` | `du -ah . \| sort -rh \| head -n 10` |

## Packages and updates

| Task | Windows | Linux |
|------|---------|-------|
| Install a package | `winget install name`; `choco install name` | `sudo apt install name`; `sudo dnf install name` |
| Remove a package | `winget uninstall name` | `sudo apt remove name`; `sudo dnf remove name` |
| List installed | `winget list`; `Get-Package` | `apt list --installed`; `dnf list installed` |
| Update everything | `winget upgrade --all` | `sudo apt update && sudo apt upgrade`; `sudo dnf upgrade` |
| Pending reboot | `Get-Item 'HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\WindowsUpdate\Auto Update\RebootRequired'` | `test -f /var/run/reboot-required`; `dnf needs-restarting -r` |

## Logs and events

| Task | Windows | Linux |
|------|---------|-------|
| Recent system log | `Get-WinEvent -LogName System -MaxEvents 50` | `journalctl -n 50`; `sudo tail /var/log/syslog` |
| Errors only | `Get-WinEvent -FilterHashtable @{LogName='System'; Level=2}` | `journalctl -p err -b` |
| Since a time | `Get-WinEvent -FilterHashtable @{LogName='Application'; StartTime=(Get-Date).AddHours(-1)}` | `journalctl --since "1 hour ago"` |
| Auth events | `Get-WinEvent -FilterHashtable @{LogName='Security'; Id=4624,4625}` | `journalctl -u sshd`; `sudo tail /var/log/auth.log` |
| Export a log | `wevtutil epl System C:\temp\system.evtx` | `journalctl -u nginx -o export > nginx.journal` |

## Scheduling

| Task | Windows | Linux |
|------|---------|-------|
| List scheduled jobs | `schtasks /Query`; `Get-ScheduledTask` | `crontab -l`; `systemctl list-timers` |
| Create a daily job | `schtasks /Create /SC DAILY /TN name /TR cmd /ST 02:00`; `Register-ScheduledTask` | `crontab -e` then `0 2 * * * cmd`; a systemd timer unit |
| Run a job now | `schtasks /Run /TN name`; `Start-ScheduledTask name` | `systemctl start name.service` |

## Remote management

| Task | Windows | Linux |
|------|---------|-------|
| Interactive shell | `Enter-PSSession host.example.com`; `mstsc /v:host.example.com` | `ssh user@host.example.com` |
| Run a command remotely | `Invoke-Command -ComputerName host.example.com -ScriptBlock { cmd }` | `ssh user@host.example.com 'cmd'` |
| Copy files | `Copy-Item -ToSession (New-PSSession host.example.com) src dst`; `robocopy src \\host.example.com\share` | `scp src user@host.example.com:dst`; `rsync -av src user@host.example.com:dst` |
| Enable remote management | `Enable-PSRemoting -Force` | `sudo systemctl enable --now ssh` |
| Reboot a remote host | `shutdown /r /m \\host.example.com /t 0`; `Restart-Computer -ComputerName host.example.com` | `ssh user@host.example.com 'sudo reboot'` |

## Permissions

| Task | Windows | Linux |
|------|---------|-------|
| Show permissions | `icacls file`; `Get-Acl file \| Format-List` | `ls -l file`; `getfacl file` |
| Grant access | `icacls dir /grant "EXAMPLE\grp:(OI)(CI)M"` | `chmod g+rwX dir`; `setfacl -m g:grp:rwX dir` |
| Change owner | `takeown /F file`; `icacls file /setowner name` | `sudo chown user:group file` |
| Reset to inherited permissions | `icacls dir /reset /T` | `chmod -R g+rX dir` (no inheritance model; use default ACLs) |
| Elevate | `Start-Process pwsh -Verb RunAs` | `sudo cmd`; `sudo -i` |

## PowerShell to bash idiom notes

- **Objects vs text.** PowerShell pipes typed objects, so `Get-Process | Sort-Object CPU` sorts on a property. Bash pipes bytes, so `ps aux | sort -k3 -nr` sorts on column 3 and you must know the column layout and watch for fields that contain spaces.
- **`Select-Object` vs `awk` and `cut`.** `Select-Object Name, Id` picks properties by name. `awk '{print $1, $2}'` or `cut -d: -f1,3` picks columns by position; `awk` handles variable whitespace, `cut` needs one fixed delimiter.
- **`Where-Object` vs `grep`.** `Where-Object { $_.Status -eq 'Running' }` filters on a property value. `grep` filters whole lines on a pattern; `awk '$3 == "running"'` filters on a column.
- **`ForEach-Object` vs `xargs` and loops.** `... | ForEach-Object { Do-Thing $_ }` becomes `... | xargs -I{} do-thing {}` or `while read -r x; do do-thing "$x"; done`.
- **Exit codes and errors.** PowerShell has `$?` (boolean) and `$LASTEXITCODE`; bash has `$?` (integer, 0 is success). Use `set -euo pipefail` in bash scripts so a failing step stops the run.
- **Quoting and case.** Both expand `$var` only inside double quotes, but PowerShell escapes with a backtick and bash with a backslash; bash uses `$(cmd)` for command substitution. PowerShell comparisons are case-insensitive by default (`-ceq` for sensitive); Linux tools are case-sensitive unless told otherwise (`grep -i`).

---

**Author:** Danny Stanfield · Perth, WA  
**License:** MIT
