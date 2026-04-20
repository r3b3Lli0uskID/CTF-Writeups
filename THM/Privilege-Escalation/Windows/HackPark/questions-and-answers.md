# HackPark — THM Room Questions

> **Room**: https://tryhackme.com/room/hackpark
> Fill in answers as you complete each task. Update [[README]] checklist when done.

---

## Task 2 — Using Hydra to Brute-Force a Login

**Q1. What request type is the Windows website login form using?**

```
POST
```

**Q2. Gain credentials to a user account! (Hydra command + result)**

```
hydra -l admin -P /usr/share/wordlists/rockyou.txt 10.49.153.22 http-post-form "/Account/login.aspx:__VIEWSTATE=<value>&__EVENTVALIDATION=<value>&ctl00$MainContent$LoginUser$UserName=^USER^&ctl00$MainContent$LoginUser$Password=^PASS^&ctl00$MainContent$LoginUser$LoginButton=Log+in:Login failed"

Answer: 1qaz2wsx
```

---

## Task 3 — Compromise the Machine

**Q1. What is the version of BlogEngine?**

```
3.3.6.0
```

**Q2. What is the CVE? (Format: CVE-YEAR-NUMBER)**

```
CVE-2019-6714
```

**Q3. Who is the webserver running as?**

```
iis apppool\blog
```

---

## Task 4 — Windows Privilege Escalation (via Metasploit)

> *No answer needed for Steps 1 & 2 (generate meterpreter payload)*

**Q1. What is the OS version of this Windows machine?**

```
Windows 2012 R2 (6.3 Build 9600)
```

![[Pasted image 20260308194148.png]]
![[Pasted image 20260308194915.png]]

**Q2. What is the name of the service running an automated task?**

```
WindowsScheduler
```

**Q3. What is the name of the binary you're supposed to exploit?**

```
Message.exe
```

**Q4. What is the user flag (on Jeff's Desktop)?**

```
759bd8af507517bcfaede78a21a73e39
```

**Q5. What is the root flag?**

```
7e13d97f05f7ceb9881a3eb3d78d3e72
```

---

## Task 5 — Privilege Escalation Without Metasploit

> *No answer needed for Steps 1 & 2 (generate shell payload + pull to box)*

**Q1. Using WinPEAS, what was the Original Install time? (Format: M/D/YYYY, HH:MM:SS AM/PM)**

```
8/3/2019, 10:43:23 AM
```

---

## Quick Answer Cheatsheet

| Task | Question | Answer |
|---|---|---|
| 2 | Login form request type | `POST` |
| 2 | Cracked credentials | `admin : 1qaz2wsx` |
| 3 | BlogEngine version | `3.3.6.0` |
| 3 | CVE | `CVE-2019-6714` |
| 3 | Webserver running as | `iis apppool\blog` |
| 4 | OS Version | `Windows 2012 R2 (6.3 Build 9600)` |
| 4 | Vulnerable service | `WindowsScheduler` |
| 4 | Exploit binary | `Message.exe` |
| 4 | User flag | 759bd8af507517bcfaede78a21a73e39 |
| 4 | Root flag | 7e13d97f05f7ceb9881a3eb3d78d3e72 |
| 5 | WinPEAS install time | 8/3/2019, 10:43:23 AM |

---

## Related

- [[HackPark/README]] — checklist and attack chain
- [[HackPark/Manual-Steps]] — step-by-step execution guide
- [[HackPark/VAPT-Report]] — full pentest report (update flags here too)
