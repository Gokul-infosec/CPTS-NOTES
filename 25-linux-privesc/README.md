# Linux Privilege Escalation

> Status: 🟨 In progress
> Date started: 2026-10-26
> Date finished: 

## 1. Goal
Escalate from a low-priv shell to root on Linux by finding misconfigurations. The enumeration checks below are *both* the attacker's privesc hunt and the defender's hardening audit — same checks, opposite goals.

## Privilege-escalation enumeration (quick checks)
Run these right after landing a shell (automate with linpeas.sh / LinEnum):
```bash
id ; sudo -l                               # my rights + what I can run as root (biggest win)
find / -perm -4000 2>/dev/null             # SUID binaries (check GTFOBins)
find / -perm -2000 2>/dev/null             # SGID
find / -perm -o+w -type f 2>/dev/null      # world-writable files
find / -perm -o+w -type d 2>/dev/null      # world-writable dirs
crontab -l ; sudo crontab -l ; cat /etc/crontab ; ls -la /etc/cron.*   # cron jobs (writable script = root)
cat ~/.bash_history ; ls -la ~             # creds/history
cat /etc/passwd ; cat /etc/group           # users/groups
ldd <binary>                               # hijackable shared libs
sestatus                                   # SELinux on/off
cat /etc/os-release ; uname -a             # version → kernel exploits
```
> `sudo -l`, writable cron scripts, and abusable SUID (via **GTFOBins**) are the three most common quick wins. Always check these first.

## Linux hardening (defense / report side)
- **Patching:** automate — `unattended-upgrades` (Debian/Ubuntu), `yum-cron`/`dnf-automatic` (RHEL).
- **Config:** audit SUID/SGID + world-writable; absolute paths in cron/sudo; no creds in files/history; remove unused packages/services; SELinux or AppArmor enforcing.
- **Users:** least privilege for sudo, limit admins, strong password policy (`/etc/login.defs`), monitor `/var/log/auth.log`.
- **Audit:** baseline against **DISA STIGs** / CIS Benchmarks; map to ISO 27001 / PCI-DSS / HIPAA; run **Lynis** (`./lynis audit system` → hardening index + suggestions).
- **Beyond:** file-integrity monitoring, centralized logging, kernel/network hardening, container hardening, security-awareness training.

> Same table, two readers: an attacker runs `find / -perm -4000` to exploit SUID; a defender runs it to remove the unneeded ones. Knowing both makes the privesc finding *and* its fix easy to write up.

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
