# TryHackMe Writeups

Hi, I'm Fiza. I'm currently learning cybersecurity through TryHackMe and this repo is where I document everything I work through. Each writeup covers my thought process, the tools I used, and what I actually learned from the room — not just the answers.

I'm still early in my journey (just finishing the Pre-Security path) so if you're a complete beginner looking for writeups that actually explain things without assuming you already know everything, you're in the right place.

My TryHackMe profile: [fiza.sk293](https://tryhackme.com/p/fiza.sk293)

---

## Stats

|---|---|
| Rank | Top 35% |
| Completed Rooms | 16 |
| Badges | 4 |
| Current Path | Pre-Security |

---

## Writeups

| Room | Difficulty | Category | Date |
|------|-----------|---------|------|
| [IDE](./ide/ide-writeup-full.md) | Easy | Linux, Enumeration, Privilege Escalation | 28-04-2026 |

More coming as I work through rooms.

---

## How I Approach These

Every writeup follows roughly the same structure:

1. Scan the target and see what's running
2. Enumerate everything before trying to exploit anything
3. Get a foothold
4. Escalate privileges
5. Document what I learned and what tripped me up

I try to write these in a way that my past self would have found useful — clear, honest about where I got stuck, and focused on understanding rather than just getting the flags.

---

## Tools I Use

| Tool | What For |
|------|---------|
| `nmap` | Port scanning and service detection |
| `gobuster` / `ffuf` | Directory and file enumeration |
| `searchsploit` | Finding known exploits |
| `netcat` | Reverse shell listeners |
| `linpeas` | Automated privilege escalation enumeration |
| THM AttackBox | Browser-based attack machine |

---

## Note on Spoilers

I try to explain the reasoning behind each step rather than just dumping commands and flags. For rooms that are still active on TryHackMe, I avoid giving away answers outright — the goal is that reading this helps you understand the approach, not just copy the solution.

---

*Updated as I complete more rooms. Feel free to open an issue if something in a writeup is wrong or unclear.*
