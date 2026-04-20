# Mr. Robot CTF — Questions & Answers

> **Room**: <https://tryhackme.com/room/mrrobot>

---

## Task 1 — Deploy the machine

> *No questions — just deploy and connect.*

Target IP (sample): `10.49.164.182`

---

## Task 2 — What is key 1?

### Question 2.1

What is key 1?

```
073403c8a58a1f80d943455fb30724b9
```

> Found in `/robots.txt` → `key-1-of-3.txt`

---

## Task 3 — What is key 2?

### Question 3.1

What is key 2?

```
822c73956184f694993bede3eb39f959
```

> Found in `/home/robot/key-2-of-3.txt` after cracking `robot:c3fcd3d76192e4007dfb496cca67e13b` (MD5 of "abcdefghijklmnopqrstuvwxyz") via WordPress theme-editor RCE.

---

## Task 4 — What is key 3?

### Question 4.1

What is key 3?

```
04787ddef27c3dee1ee161b21670b4e4
```

> Found in `/root/key-3-of-3.txt` after exploiting **SUID Nmap** with `nmap --interactive` → `!sh` → root shell.

---

## Quick Answer Cheatsheet

| Task | Question | Answer |
|------|----------|--------|
| 2 | Key 1 | `073403c8a58a1f80d943455fb30724b9` |
| 3 | Key 2 | `822c73956184f694993bede3eb39f959` |
| 4 | Key 3 | `04787ddef27c3dee1ee161b21670b4e4` |
