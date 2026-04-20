# Mr. Robot CTF — TryHackMe

> **Room**: https://tryhackme.com/room/mrrobot
> **Difficulty**: Medium
> **OS**: Linux
> **Status**: Complete ✓

---

## Overview

A Mr. Robot themed boot-to-root machine. Three hidden keys (flags) must be found across the system. The attack chain involves WordPress enumeration, brute force, authenticated RCE via theme editor, hash cracking, and SUID privilege escalation.

---

## Attack Chain Summary

```
Recon (nmap) → Web Enum (robots.txt, Gobuster) → Key 1 (robots.txt)
→ WordPress brute force (fsocity.dic) → Theme RCE → Shell as daemon
→ Key 2 (/home/robot — after cracking MD5 hash) → SUID nmap → Root → Key 3
```

---

## Checklist

### Phase 1 — Recon
- [ ] nmap full port scan
- [ ] HTTP/HTTPS service fingerprinting
- [ ] Web directory enumeration (Gobuster)
- [ ] robots.txt review

### Phase 2 — Initial Foothold
- [ ] Retrieve Key 1 from web
- [ ] Download `fsocity.dic` wordlist
- [ ] Identify WordPress version and login page
- [ ] Brute force WordPress login (WPScan / Hydra)
- [ ] Authenticate to WordPress admin
- [ ] Edit theme PHP for reverse shell
- [ ] Catch shell → daemon user

### Phase 3 — Privilege Escalation to robot
- [ ] Find `/home/robot/password.raw-md5`
- [ ] Crack MD5 hash (hashcat / crackstation)
- [ ] `su robot` → read Key 2

### Phase 4 — Privilege Escalation to root
- [ ] Enumerate SUID binaries
- [ ] Exploit nmap interactive mode → root shell
- [ ] Read Key 3 from /root/

---

## Flags

| Key   | Location                         | Value |
|-------|----------------------------------|-------|
| Key 1 | `/key-1-of-3.txt` (web root)     | `073403c8a58a1f80d943455fb30724b9` |
| Key 2 | `/home/robot/key-2-of-3.txt`     | `822c73956184f694993bede3eb39f959` |
| Key 3 | `/root/key-3-of-3.txt`           | `04787ddef27c3dee1ee161b21670b4e4` |

---

## Target Info

| Field       | Value              |
|-------------|--------------------|
| IP          | 10.49.164.182      |
| OS          | Linux              |
| Web         | Apache + WordPress |
| Open Ports  | 80, 443            |

---

## Related

- [[MrRobot/THM-Questions]] — room answers
- [[MrRobot/Manual-Steps]] — team walkthrough guide
- [[MrRobot/VAPT-Report]] — pentest report
