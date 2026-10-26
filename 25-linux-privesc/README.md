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

## Deep enumeration (what to collect)
Beyond the quick checks — automate with **linpeas.sh** but know the manual commands:
```bash
# environment
whoami; id; hostname; ip a; env; cat /etc/os-release; echo $PATH
# defenses
iptables -L; sestatus; apparmor_status; ufw status; fail2ban-client status
# users / login
cat /etc/passwd; cat /etc/group; getent group sudo; lastlog; w; history
# drives / shares / mounts
lsblk; cat /etc/fstab; showmount -e <host>; cat /etc/exports
# processes (catch root cron/jobs live)
ps aux | grep root
./pspy64 -pf -i 1000            # watch processes/cron without root
# installed packages vs GTFOBins (auto-flag abusable binaries)
apt list --installed 2>/dev/null | cut -d/ -f1 > pkgs.list
# cross-ref pkgs.list against gtfobins.github.io
```

### Credential hunting (where secrets hide)
```bash
grep -riE 'password|passwd|user|secret|api[_-]?key' /var/www /etc /opt /home 2>/dev/null
find / -type f \( -name "*.conf" -o -name "*.config" -o -name "*.bak" -o -name "*.xml" -o -name "*.sql" \) 2>/dev/null
grep -iE 'password|pass' ~/.bash_history
# SSH keys
find / -name id_rsa 2>/dev/null ; ls -la ~/.ssh ; cat ~/.ssh/known_hosts
```

## Privilege-escalation techniques (reference)
| Vector | How | 
|---|---|
| **sudo -l** | any allowed binary → check **GTFOBins** for the root-shell trick; `sudo <bin>` |
| **SUID/SGID** | `find / -perm -4000` → GTFOBins the binary |
| **sudo tcpdump** | `-z` postrotate runs a script as root |
| **Writable cron script** | edit the script a root cron runs |
| **PATH hijack** | writable dir in PATH + a script calling a bare binary → drop a malicious one; `PATH=.:$PATH` |
| **LD_PRELOAD** | `sudo LD_PRELOAD=/tmp/x.so <allowed cmd>` loads your lib as root (if env_keep) |
| **Shared-library hijack** | `ldd <bin>` / `readelf -d <bin>|grep PATH` → writable RUNPATH → drop a malicious `.so` |
| **LXD/LXC group** | import image → privileged container → mount host `/` → read root |
| **NFS no_root_squash** | mount the share, create a SUID root binary on it, run it on the target |
| **Kernel exploit** | `uname -a` → matching CVE (e.g. DirtyPipe/DirtyCow); `gcc exploit.c -o x` — last resort, can crash the box |

> Order of attack: `sudo -l` → SUID+GTFOBins → cron → writable files/PATH → then the exotic stuff (LD_PRELOAD, LXD, NFS) → kernel exploit last (riskiest). pspy is the secret weapon — it reveals root cron jobs and processes you can't see otherwise.

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
