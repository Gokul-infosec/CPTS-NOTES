# CPTS Exam Tips

Difficulty: roughly between OSCP and OSEP, but with more guidance. (Tips also broadly apply to CBBH / CDSA / CWEE.)

## Exam structure
- **Capture 12 of 14 flags to pass.** Once you have a flag, grab **all proof/screenshots before digging deeper** — don't lose evidence by moving on.
- **Report is long** — budget up to ~2 days just for writing. Make sure you have the free time blocked.

## Skills to have cold before exam day
- **Port forwarding & pivoting** (module 12): move to/from hosts, bypass inbound/outbound firewalls, reach isolated networks through chained hosts. This is heavily tested.
- **Managing multiple sessions** at once — e.g. a low-priv and a SYSTEM session per host; switch contexts smoothly (Meterpreter / Sliver / shells).
- **Persistence both ways** — victim→me (services, startup, scheduled tasks) and me→victim (SSH keys, RDP, local admin), so a dropped VPN doesn't lose access. Keep sessions stable.

## Preparation
- **Keep a record of payloads + notes that worked** to get access. Test Mimikatz, Rubeus, PowerView, pivoting, Meterpreter *in the labs* — don't learn their features on exam day.
- Build a **personal cheat sheet** (this repo) for quick reference.
- **Keep local copies of exploits that actually worked** in your labs — online PoCs often don't work for many reasons; a verified local copy saves you.

## My exam-prep checklist
- [ ] Pivoting fluent (Ligolo-ng + SSH tunnels + chisel)
- [ ] Multi-session handling practiced
- [ ] Persistence both directions practiced
- [ ] Core AD attacks run manually (not just via MSF): LLMNR, Kerberoast, AS-REP, DCSync, PtH
- [ ] Linux + Windows privesc checklists memorized
- [ ] Report template ready (Sysreptor), time blocked for writing
