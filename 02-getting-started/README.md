# Getting Started

> Status: 🟨 In progress
> Date started: 2026-10-26
> Date finished: 

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

## 2. Setup
- Attack box: HTB Pwnbox (Parrot), unlimited time
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
