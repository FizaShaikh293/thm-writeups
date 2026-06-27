# 🛡️ TryHackMe Writeups

## Hi, I'm Fiza 🙋‍♀️

I'm currently learning cybersecurity through TryHackMe, and this repository is where I document everything I work through.

Each writeup includes my thought process, tools used, and key lessons learned.

**TryHackMe Profile:**
[fiza.sk293](https://tryhackme.com/p/fiza.sk293)

---

# 📊 Stats

* **Rank:** Top 9%
* **Completed Rooms:** 44
* **Badges:** 11

---

# 📝 Writeups

## 🔹 [IDE](https://fizashaikh293.github.io/thm-writeups/ide/ide-writeup-full.html)

* Difficulty: Easy
* Category: Linux, Enumeration, Privilege Escalation
* Date: 28-04-2026

---

## 🔹 [AI Threat Modelling Assessment](https://fizashaikh293.github.io/thm-writeups/AI%20Threat%20Modelling/AI-TM_Writeup.html)

* Difficulty: Easy
* Category: AI Security, Threat Modelling
* Date: 22-05-2026

---

## 🔹 [Pre Security Path](https://fizashaikh293.github.io/thm-writeups/Pre%20Security%20Path/)

* Difficulty: Easy
* Category: Pre Security
* Date: 06-06-2026

---

## 🔹 [TryHack3M: Bricks Heist](https://fizashaikh293.github.io/thm-writeups/Bricks%20Heist/full_writeup.html)

* Difficulty: Easy
* Category: Boot-to-Root CTF + Threat Hunting
* Date: 08-06-2026

---

## 🔹 [Pickle Rick](https://fizashaikh293.github.io/thm-writeups/Pickle%20Rick/pickle-rick_writeup.html)

* Difficulty: Easy
* Category: Linux, Web Exploitation, Enumeration, Privilege Escalation
* Date: 10-06-2026

---

## 🔹 [Wonderland](https://fizashaikh293.github.io/thm-writeups/Wonderland/wonderland_writeup.html)

* Difficulty: Medium
* Category: Linux, Enumeration, Privilege Escalation
* Date: 12-06-2026

---

# 🔍 How I Approach Writeups

1. Scan the target and identify running services
2. Enumerate everything before exploiting anything
3. Gain initial access (foothold)
4. Escalate privileges
5. Document findings clearly

I focus on understanding *why* something works, not just getting the flag.

---

# 🛠️ Tools I Use

* `nmap` → port scanning & service detection
* `gobuster` / `ffuf` → directory enumeration
* `searchsploit` → exploit lookup
* `netcat` → reverse shells
* `linpeas` → privilege escalation enumeration
* `strings` / `ghidra` → binary analysis
* `getcap` → Linux capabilities
* `GTFOBins` → privilege escalation techniques
* `THM AttackBox` → browser-based hacking environment

---

# 🎯 Techniques Covered

* Command injection via web shell
* Sudo misconfiguration (`NOPASSWD: ALL`)
* Sensitive data exposure in HTML / robots.txt
* Python library hijacking
* SUID PATH hijacking
* Linux capabilities exploitation (`cap_setuid`)
* Hidden CSS credential discovery

---

# 📝 Note on Writeups

These are written for learning purposes and are not meant to be copy-paste solutions.

Each writeup focuses on:

* Understanding the attack path
* Why each step works
* How to think like an attacker

---

*Updated regularly as I complete more TryHackMe rooms.*
