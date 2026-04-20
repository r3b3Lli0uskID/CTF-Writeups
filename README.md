# CTF Write-ups & VAPT Reports

Cybersecurity challenge walkthroughs and penetration testing reports from TryHackMe, HackTheBox, and other platforms.

Each room folder contains the same five-document deliverable, designed for review *and* for first-time-learner study:

| Doc | Purpose |
| --- | --- |
| `README.md` | Quick index — room link, category, difficulty, flag, file map |
| `writeup.md` | Narrative writeup — full reasoning, including trial-and-error journey when interesting |
| `beginner-walkthrough.md` | Step-by-step manual a first-timer can follow command-by-command |
| `VAPT-report.md` | Formal Vulnerability Assessment & Pen-Test report (executive summary, findings with CVSS, MITRE ATT&CK mapping, recommendations) |
| `questions-and-answers.md` | Clean Q&A list as submitted to the platform |

Plus optional `evidence/` and `driver/` (or similar) sub-folders for scripts and source code we built.

---

## Layout

Rooms are categorized **by primary skill exercised**, not by the platform's official tag:

```
THM/                              # TryHackMe
├── Privilege-Escalation/
│   ├── Windows/
│   │   └── HackPark/
│   └── Linux/
│       └── MrRobot/
└── Malware/
    ├── Static-Analysis-DFIR/
    │   └── Masquerade/
    └── Rootkit-Development/
        └── Kernel-Blackout/

HTB/                              # (HackTheBox — coming as we play)
```

---

## Index

### TryHackMe

| Room | Category | Difficulty | Key Techniques | Flag | Status |
| --- | --- | --- | --- | --- | --- |
| [HackPark](THM/Privilege-Escalation/Windows/HackPark/) | Windows priv-esc | Medium | Hydra brute force, CVE-2019-6714 (BlogEngine.NET RCE), WinPEAS, service binary hijack | `759bd8af…` / `7e13d97f…` | ✅ Complete |
| [Mr. Robot](THM/Privilege-Escalation/Linux/MrRobot/) | Linux priv-esc | Medium | WordPress enum, WPScan brute force, theme editor RCE, MD5 cracking, SUID Nmap | (3 keys) | ✅ Complete |
| [Masquerade](THM/Malware/Static-Analysis-DFIR/Masquerade/) | DFIR / static malware | Medium | EVTX ScriptBlock 4104, RC4 + AES-CBC C2 decode, .NET UTF-16 string analysis, MITRE mapping | `THM{m45k3d_tr4ff1c_0v3r_c0v3rt_ch4nn3lz}` | ✅ Complete |
| [Kernel Blackout](THM/Malware/Rootkit-Development/Kernel-Blackout/) | Windows kernel rootkit | Hard | DKOM `ActiveProcessLinks` unlink via kdmapper-loaded driver, WinRM build pipeline, brute-force EPROCESS offset discovery | `THM{H1D3N_FR0M_US3RSP4C3}` | ✅ Complete |

### HackTheBox

| Box | OS | Difficulty | Key Techniques | Status |
| --- | --- | --- | --- | --- |
| *Coming soon* | | | | |

---

## Stats

| Metric | Count |
| --- | --- |
| Rooms completed | 4 |
| Platforms | TryHackMe |
| Categories covered | Windows priv-esc, Linux priv-esc, DFIR/Static malware, Kernel rootkit dev |
| Total flag captures | 7+ (multi-flag rooms counted once each) |
| Trial-and-error iterations documented | 16 (Kernel Blackout) |

---

## Methodology

All engagements follow the standard penetration-testing methodology:

```
Reconnaissance → Scanning → Enumeration → Exploitation → Post-Exploitation → Reporting
```

Tools used across the portfolio: `nmap`, `hydra`, `wpscan`, `gobuster`, `metasploit`, `WinPEAS`, `LinPEAS`, `hashcat`, `john`, `burpsuite`, `netcat`, `nxc` (NetExec), `tshark`, `python-evtx`, `pycryptodome`, `xfreerdp3`, VS 2022 BuildTools (`cl.exe`, `link.exe /DRIVER`), Windows 10 SDK + WDK, hand-rolled C and Python.

---

## Template

Use [`templates/writeup-template.md`](templates/writeup-template.md) as a starting point for new write-ups. The five-doc deliverable above is the standard for each room.

---

## Disclaimer

All activities documented here were performed in **authorized lab environments** (TryHackMe, HackTheBox). Real malware artifacts are gitignored — reproduce by downloading the room's task files yourself. Never test systems without explicit written permission.
