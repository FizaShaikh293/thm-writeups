# TryHackMe Writeups
---
## Hi, I'm Fiza 🙋‍♀️
I'm currently learning cybersecurity through TryHackMe and this repo is where I document everything I work through. Each writeup covers my thought process, the tools I used and what I actually learned from the room.

My TryHackMe profile: [fiza.sk293](https://tryhackme.com/p/fiza.sk293)

---
## Stats
| Stat | Value |
|------|-------|
| Rank | Top 9% |
| Completed Rooms | 44 |
| Badges | 11 |

---
## Writeups
| Room/Learning Path | Difficulty | Category | Date |
|------|------------|----------|------|
| [IDE](https://fizashaikh293.github.io/thm-writeups/ide/ide-writeup-full.html) | Easy | Linux, Enumeration, Privilege Escalation | 28-04-2026 |
| [AI Threat Modelling Assessment](https://fizashaikh293.github.io/thm-writeups/AI%20Threat%20Modelling/AI-TM_Writeup.html) | Easy | AI Security, Threat Modelling | 22-05-2026 |
| [Pre Security Path](https://fizashaikh293.github.io/thm-writeups/Pre%20Security%20Path/) | Easy | Pre Security | 06-06-2026 |
| [TryHack3M: Bricks Heist](https://fizashaikh293.github.io/thm-writeups/Bricks%20Heist/full_writeup.html) | Easy | Boot-to-Root CTF + Threat Hunting | 08-06-2026 |
| [Pickle Rick](https://fizashaikh293.github.io/thm-writeups/Pickle%20Rick/pickle-rick_writeup.html) | Easy | Linux, Web Exploitation, Enumeration, Privilege Escalation | 10-06-2026 |
| [Wonderland](https://fizashaikh293.github.io/thm-writeups/Wonderland/wonderland_writeup.html) | Medium | Linux, Enumeration, Privilege Escalation | 12-06-2026 |

More coming as I work through Rooms and Learning Paths.

---
## How I Approach These
Every writeup follows roughly the same structure:

- Scan the target and see what's running
- Enumerate everything before trying to exploit anything
- Get a foothold
- Escalate privileges
- Document what I learned and what tripped me up

I try to write these in a way that my past self would have found useful: clear, honest about where I got stuck and focused on understanding rather than just getting the flags.

---
## Tools I Use
| Tool | What For |
|------|----------|
| nmap | Port scanning and service detection |
| gobuster / ffuf | Directory and file enumeration |
| searchsploit | Finding known exploits |
| netcat | Reverse shell listeners |
| linpeas | Automated privilege escalation enumeration |
| strings / ghidra | Binary analysis |
| getcap | Enumerating Linux capabilities |
| GTFOBins | Capability and SUID exploit one-liners |
| THM AttackBox | Browser-based attack machine |

## Techniques Covered
| Technique | Seen In |
|-----------|---------|
| Command injection via web shell | Pickle Rick |
| Sudo misconfiguration (NOPASSWD: ALL) | Pickle Rick |
| Credential exposure in HTML source and robots.txt | Pickle Rick |
| Python library hijacking | Wonderland |
| SUID binary PATH hijacking | Wonderland |
| Linux capabilities exploit (cap_setuid) | Wonderland |
| CSS hidden credential extraction | Wonderland |

---
## Note on Answers
These writeups include task answers and flags. The goal is not to hand you something to copy paste but to walk through the reasoning so you understand why each answer is correct. If you are using these to check your own work or get unstuck on a specific step that is exactly what they are here for.

---
*Updated as I complete more rooms. Feel free to open an issue if something in a writeup is wrong or unclear.*
