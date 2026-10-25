# Footprinting

> Status: 🟨 In progress
> Date started: 2026-10-26
> Date finished: 

## 1. Goal
Per-service footprinting/enumeration. (Service-command reference also in `cheatsheets/service-enumeration.md`.)

## LDAP (389/636) — enumeration & injection (reference)
LDAP = directory service (users, groups, devices) in a tree (DIT) of DNs with attributes (cn, uid, mail). Ports **389** (plain) / **636** (LDAPS). Implementations: OpenLDAP, Microsoft AD (AD uses LDAP).

**Enumeration**
```bash
nmap -p389,636 -sV -sC <HOST>
# anonymous bind (some servers allow read):
ldapsearch -x -H ldap://<HOST> -b "<base_dn>" "(objectClass=*)"
# authenticated bind:
ldapsearch -x -H ldap://<HOST> -D "<bind_dn>" -w "<pass>" -b "<base_dn>" "(objectClass=*)"
enum4linux -a <HOST>            # broader Windows/AD enum
```
- Try common base DNs (`dc=example,dc=com`, `ou=users,...`), look for `sAMAccountName`, enumerate groups.
- GUI browsers: JXplorer, Apache Directory Studio, LdapAdmin.

**LDAP injection** (app builds an LDAP filter from user input — like SQLi for LDAP). Special chars: `*` `(` `)` `|` `&` `!` `\`.
- Auth bypass example: filter `(&(uid=$u)(userPassword=$p))`, enter username `*)(uid=*` to break the logic.
- Enumerate users: `*)(|(cn=*)(mail=*))` · privilege probe: `admin*)(|(objectClass=*))`
- **Fix:** sanitize/escape LDAP special chars, parameterize, least-privilege bind account, LDAPS, WAF.

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
