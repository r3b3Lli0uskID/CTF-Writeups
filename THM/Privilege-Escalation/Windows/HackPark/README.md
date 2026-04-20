# Lab: HackPark

> **Platform**: TryHackMe
> **Type**: `lab`
> **URL**: https://tryhackme.com/room/hackpark
> **Status**: Complete

## Machine Details

| IP | OS | Difficulty | Theme |
|---|---|---|---|
| 10.49.153.22 | Windows Server 2012 R2 (6.3.9600) | Medium | BlogEngine.NET |

## Attack Chain

1. **Hydra** — brute force BlogEngine.NET admin login at `/Account/login.aspx`
2. **CVE-2019-6714** — BlogEngine.NET path traversal → RCE via malicious `.ascx` upload
3. **WinPEAS** — enumerate privesc vectors
4. **SystemScheduler** — weak service binary → replace with reverse shell → SYSTEM

## Checklist

- [x] VPN connected (THM AP South 1)
- [x] Nmap scan complete
- [x] Web app identified — BlogEngine.NET on IIS 8.5
- [x] Hydra — crack admin password
- [x] Login to BlogEngine.NET admin panel
- [x] CVE-2019-6714 — upload malicious `.ascx` reverse shell
- [x] Catch reverse shell → user foothold
- [x] Upload & run WinPEAS
- [x] Identify privesc vector (SystemScheduler)
- [x] Replace autorun binary → catch SYSTEM shell
- [x] Grab user.txt
- [x] Grab root.txt

## Recon

### Nmap Results
```
PORT     STATE SERVICE       VERSION
80/tcp   open  http          Microsoft IIS httpd 8.5
  robots.txt entries: /Account/*.* /search /search.aspx /error404.aspx /archive
  Title: hackpark | hackpark amusements
3389/tcp open  ms-wbt-server Microsoft Terminal Services
  Hostname: HACKPARK
  OS: Windows (Server 2012 R2 — build 6.3.9600)
```

### Web App
- **App**: BlogEngine.NET
- **Server**: Microsoft IIS 8.5 + ASP.NET
- **Login**: `http://10.49.153.22/Account/login.aspx`
- **Username**: `admin` (default for BlogEngine)
- **Form fields**:
  - `ctl00$MainContent$LoginUser$UserName`
  - `ctl00$MainContent$LoginUser$Password`

## Foothold — CVE-2019-6714

**Vulnerability:** BlogEngine.NET 3.3.6 path traversal via `theme` parameter allows uploading `.ascx` webshell
**Steps:**
1. Login with cracked admin creds
2. Upload malicious `PostView.ascx` to `/App_Data/files/`
3. Trigger via `?theme=../../App_Data/files`
4. Catch reverse shell on Kali listener

## Privilege Escalation — SystemScheduler

**Vector:** Weak permissions on `WindowsScheduler` service autorun binary
**Steps:**
1. Run WinPEAS — identify `message.exe` writeable by user
2. Replace `message.exe` with msfvenom reverse shell payload
3. Wait for scheduler to execute → catch SYSTEM shell

## Flags

| Flag | Value |
|---|---|
| User (user.txt) | 759bd8af507517bcfaede78a21a73e39 |
| Root (root.txt) | 7e13d97f05f7ceb9881a3eb3d78d3e72 |

## Key Takeaways

- ASP.NET VIEWSTATE is static enough for Hydra to brute force
- BlogEngine.NET CVE-2019-6714 is a classic path traversal → RCE
- WinPEAS autorun check is critical on Windows boxes

## References

- [[Pentesting/Cheatsheets/Methodology]]
- [[Pentesting/Tools]]
- CVE-2019-6714: BlogEngine.NET 3.3.6 RCE
