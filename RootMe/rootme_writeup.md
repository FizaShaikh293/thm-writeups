---
layout:default
---


# [RootMe](https://tryhackme.com/room/rrootme)

| Field      | Details                          |
|------------|----------------------------------|
| Room       | [RootMe](https://tryhackme.com/room/rrootme) |
| Platform   | TryHackMe                        |
| Difficulty | Easy                              |
| Category   | Web Exploitation, Linux PrivEsc  |
| Tags       | File Upload Bypass, Reverse Shell, SUID Abuse |

---

## Overview

The room asks a simple question. Can you root me. The answer involves a file upload form that trusts extensions more than it should and a Python binary that was left with the SUID bit set like a spare key under the doormat.

From a blue team perspective this room is a compact lesson in two of the most common findings in real world web app assessments: insufficient upload validation and forgotten SUID binaries. Neither requires exotic tooling. Both show up constantly in actual pentest reports, which is exactly why this "beginner" room earns a permanent place in any fundamentals refresher.

---

## Enumeration

### Port Scan

```bash
nmap -sC -sV <TARGET_IP>
```
