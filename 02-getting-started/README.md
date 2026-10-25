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
