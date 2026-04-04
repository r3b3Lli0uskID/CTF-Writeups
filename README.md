# CTF Write-ups & VAPT Reports

Cybersecurity challenge walkthroughs and penetration testing reports from TryHackMe, HackTheBox, and other platforms.

Each write-up includes a beginner-friendly step-by-step guide explaining **what**, **why**, and **how** — not just commands to copy-paste.

---

## Write-ups

### TryHackMe

| Room | OS | Difficulty | Key Techniques | Status |
|------|-----|-----------|----------------|--------|
| [HackPark](TryHackMe/HackPark/) | Windows Server 2012 R2 | Medium | Hydra brute force, CVE-2019-6714 (BlogEngine.NET RCE), WinPEAS, service binary hijack | Complete |
| [Mr. Robot](TryHackMe/MrRobot/) | Linux | Medium | WordPress enum, WPScan brute force, theme editor RCE, MD5 cracking, SUID Nmap privesc | Complete |

### HackTheBox

| Box | OS | Difficulty | Key Techniques | Status |
|-----|-----|-----------|----------------|--------|
| *Coming soon* | | | | |

---

## VAPT Reports

Professional-format Vulnerability Assessment and Penetration Testing reports with CVSS 3.1/4.0 scoring.

| Report | Target | Findings |
|--------|--------|----------|
| [HackPark VAPT Report](VAPT-Reports/HackPark-VAPT-Report.md) | HackPark (TryHackMe) | BlogEngine.NET RCE, weak credentials, unquoted service path |
| [Mr. Robot VAPT Report](VAPT-Reports/MrRobot-VAPT-Report.md) | Mr. Robot (TryHackMe) | WordPress RCE, exposed sensitive files, SUID misconfiguration |
| [Combined Report (PDF)](VAPT-Reports/Combined-VAPT-Report.pdf) | Both machines | Full engagement report with executive summary |

---

## Stats

| Metric | Count |
|--------|-------|
| Machines completed | 2 |
| Platforms | TryHackMe |
| VAPT Reports written | 3 |
| Techniques documented | 12+ |

---

## Methodology

All engagements follow the standard penetration testing methodology:

```
Reconnaissance → Scanning → Enumeration → Exploitation → Post-Exploitation → Reporting
```

Tools used: Nmap, Hydra, WPScan, Gobuster, Metasploit, WinPEAS, LinPEAS, Hashcat, John the Ripper, Burp Suite, Netcat.

---

## Template

Use [templates/writeup-template.md](templates/writeup-template.md) for new write-ups.

---

## Disclaimer

All activities documented here were performed in authorized lab environments (TryHackMe, HackTheBox). Never test systems without explicit written permission.
