---
layout: default
title: Attacks and Defenses
---

**Learning Path:** Pre-Security
**Module:** 07 - Attacks and Defenses
**Rooms Completed:** The CIA Triad · Cryptography Concepts · Become a Hacker · Become a Defender
**Date Completed:** June 2026
**Difficulty:** Beginner (Theory + Interactive Labs)

---

## Overview

Module 7 is the payoff for everything that came before it. After six modules building foundations in networking, the web, hardware, operating systems and software, this final module answers the question the whole path was building towards: what exactly are we protecting, how do attackers break it and what does defence actually look like in practice?

The CIA Triad gives you the framework for what security means. Cryptography Concepts shows you the primary tool used to enforce it. Become a Hacker puts you in the attacker's seat for the first time. Become a Defender shows you how defenders think about the same systems.

Completing this module means completing the Pre-Security path. Everything from here (Cybersecurity 101, Jr Penetration Tester, SOC Level 1) builds on this foundation.

---

## Room 1 - The CIA Triad

**Room Link:** https://tryhackme.com/room/theciatriad
**Format:** Reading + Interactive Scenario Exercise

### What It Covers

The CIA Triad is the foundational framework for defining what information security actually protects. It is not a checklist or a compliance framework, it is a mindset. Every security decision, from configuring a firewall rule to writing an incident response plan, can be evaluated against these three principles.

```
+-------------------------------------------------------+
|                    CIA TRIAD                          |
|                                                       |
|   CONFIDENTIALITY    INTEGRITY      AVAILABILITY      |
|   Ensuring data is   Ensuring data  Ensuring data     |
|   only accessible    is not         is accessible     |
|   to authorised      modified       when needed       |
|   individuals        without        by authorised     |
|                      permission     users             |
+-------------------------------------------------------+
```

**Confidentiality**

Confidentiality means that information is only accessible to those who are authorised to see it. It is broken by disclosure attacks. Examples: unauthorised access to medical records, credentials stolen by a keylogger, unencrypted data intercepted in transit, a misconfigured S3 bucket exposing private files to the internet.

Controls that protect confidentiality: encryption, access controls, role-based permissions, multi-factor authentication, data classification.

**Integrity**

Integrity means that data has not been altered without authorisation, and that any unauthorised alteration can be detected. It is broken by alteration attacks. Examples: an attacker modifying a financial transaction, malware tampering with system logs to cover its tracks, a man-in-the-middle attack silently modifying packets in transit.

Controls that protect integrity: hashing, digital signatures, version control, audit logs, file integrity monitoring.

**Availability**

Availability means that systems and data are accessible when authorised users need them. It is broken by destruction or denial attacks. Examples: ransomware encrypting a hospital's systems, a DDoS attack taking down a banking website, a misconfigured update bricking critical infrastructure.

Controls that protect availability: redundancy, backups, failover systems, DDoS protection, disaster recovery planning, patching.

**The DAD Triad (the attack counterpart):**

```
+------------------------------------------------+
|  CIA Triad (defender goals)                    |
|  Confidentiality  Integrity    Availability    |
|        |               |            |          |
|        v               v            v          |
|  DAD Triad (attack goals)                      |
|  Disclosure      Alteration    Destruction     |
+------------------------------------------------+
```

Every cyberattack targets one or more of these three pillars. Ransomware attacks availability (and often confidentiality). Data breaches attack confidentiality. Log tampering attacks integrity.

**Task Answers**

| Task | Question | Answer |
|---|---|---|
| Task 1 | I am ready to start | No answer needed |
| Task 2 | What ensures data is only accessible to authorised individuals? | Confidentiality |
| Task 2 | What ensures data cannot be modified without permission? | Integrity |
| Task 2 | What ensures data is accessible when needed? | Availability |
| Task 2 | CIA Triad is not just a set of definitions, it is a mindset. What type of mindset is it? | security |
| Task 3 | Complete the room | No answer needed |

---

## Room 2 - Cryptography Concepts

**Room Link:** https://tryhackme.com/room/cryptographyconcepts
**Format:** Reading + Secret Message Rescue Interactive Game

### What It Covers

Cryptography is the primary technical mechanism used to enforce confidentiality and integrity. It is how data in transit stays private, how digital signatures prove authenticity and how HTTPS works. This room covers the journey from classical ciphers to modern asymmetric encryption.

**Plaintext, Ciphertext and Keys:**

```
Encryption:
Plaintext + Key + Algorithm  -->  Ciphertext

Decryption:
Ciphertext + Key + Algorithm  -->  Plaintext

Without the key, ciphertext should be computationally
indistinguishable from random noise.
```

**The Caesar Cipher (Symmetric, Classical):**

The Caesar cipher shifts each letter in the message by a fixed number of positions in the alphabet. That fixed number is the key.

```
Key = 3

Plain:   A B C D E F G H I J K L M N O P Q R S T U V W X Y Z
Cipher:  D E F G H I J K L M N O P Q R S T U V W X Y Z A B C

HELLO  -->  KHOOR
CYBER  (key 5)  -->  HFJWJ
```

To decrypt, shift backwards by the same key. Caesar cipher is historically interesting but completely insecure: it only has 25 possible keys, making brute force trivial.

**Decoding FVZCYR PNRFNE PVCURE:**

Working through the 26 possible shifts, key 13 produces readable English: SIMPLE CAESAR CIPHER. This is ROT13, a special case where encryption and decryption use the same operation (rotating by half the alphabet).

**Symmetric vs Asymmetric Encryption:**

```
SYMMETRIC ENCRYPTION
+---+                       +---+
| A | --[shared key]-enc--> | B |
| A | <-[shared key]-dec--- | B |
+---+                       +---+
One key for both encryption and decryption.
Fast. Problem: how do you securely share the key?
Examples: AES, DES

ASYMMETRIC ENCRYPTION
+---+                              +---+
| A | --[Bob's public key]-enc---> | B |
|   | <--[Bob's private key]-dec-- |   |
+---+                              +---+
Public key encrypts. Private key decrypts.
Solves the key distribution problem.
Slower than symmetric.
Examples: RSA, ECC
```

The private key must never be shared. Alice encrypts with Bob's public key. Only Bob can decrypt with his private key. This is the core principle behind HTTPS and secure email.

**Hashing:**

Hashing converts data of any length into a fixed-length output (the hash or digest). It is a one-way function: you cannot reverse a hash to get the original input. Any change to the input, even a single bit, produces a completely different hash.

```
SHA-256("hello")  =  2cf24dba5fb0a30e...
SHA-256("Hello")  =  185f8db32921bd46...  (completely different)
```

Hashing enforces integrity. If the hash of a downloaded file matches the published hash, the file has not been tampered with in transit.

**Task Answers**

| Task | Question | Answer |
|---|---|---|
| Task 2 | Complete all levels of Secret Message Rescue. What is the flag? | (Generated after completing the game in the lab) |
| Task 2 | Using Caesar cipher with key 5, what does CYBER become? | HFJWJ |
| Task 2 | Decode FVZCYR PNRFNE PVCURE. What is the message? | SIMPLE CAESAR CIPHER |
| Task 3 | In asymmetric encryption, which key stays secret? | private key |
| Task 3 | Alice encrypts with Bob's public key. Only Bob can decrypt. Yay or Nay? | Yay |

---

## Room 3 - Become a Hacker

**Room Link:** https://tryhackme.com/room/becomeahacker
**Format:** Interactive Web App Hacking Lab (Gobuster + Manual Brute Force)

### What It Covers

This is the first room in the Pre-Security path where you actively attack something. The target is a simulated online shop. The goal: find the hidden admin login page and gain access.

Offensive security means using the same techniques as malicious attackers, but with explicit authorisation and the intention of improving defences. Ethical hackers (penetration testers, red teamers, bug bounty hunters) identify vulnerabilities before real attackers can exploit them.

**The Attack Methodology:**

```
+-----------------------------------------------+
|  OFFENSIVE SECURITY MINDSET                   |
|                                               |
|  1. RECONNAISSANCE                            |
|     Understand the target before attacking    |
|                                               |
|  2. ENUMERATION                               |
|     Find hidden pages, services, usernames    |
|                                               |
|  3. EXPLOITATION                              |
|     Use discovered information to gain access |
|                                               |
|  4. POST-EXPLOITATION                         |
|     Maintain access, pivot, find the flag     |
+-----------------------------------------------+
```

**Step 1 - Directory Enumeration with Gobuster:**

Web applications often have pages that are not linked from the main site but are still publicly accessible if you know the path. Gobuster automates the process of guessing these paths using a wordlist.

```bash
gobuster dir --url http://www.onlineshop.thm/ -w /usr/share/wordlists/dirbuster/directory-list.txt
```

Output:
```
/login    (Status: 200)
```

The hidden page is `/login`. The HTTP status 200 means it exists and is accessible.

**Step 2 - Manual Credential Brute Force:**

Navigate to `http://www.onlineshop.thm/login`. Try common username/password combinations. Username `admin` is a standard starting point. Working through common passwords:

- `abc123` - no
- `123456` - no
- `qwerty` - success

**Step 3 - The Flag:**

After logging in with `admin:qwerty`, the secret message is displayed:

```
THM{born_to_hack!}
```

**Why This Matters:**

This exercise demonstrates two real, prevalent vulnerability classes:

1. **Unprotected admin panels**: exposed login pages not linked from the main site are still accessible. Security through obscurity is not security.
2. **Weak credentials**: `admin:qwerty` would be cracked in seconds by any real attacker using automated tools like Hydra.

Both of these appear in breach reports constantly. The 2024 Verizon DBIR consistently shows stolen/weak credentials as the top initial access vector. This is not a theoretical exercise.

**Task Answers**

| Task | Question | Answer |
|---|---|---|
| Task 1 | No answer needed | - |
| Task 2 | What hidden web page did you discover? | /login |
| Task 2 | What status code is returned when accessing the hidden page? | 200 |
| Task 3 | What password did you find for the admin user? | qwerty |
| Task 3 | What secret message is displayed after logging in? | THM{born_to_hack!} |

---

## Room 4 - Become a Defender

**Room Link:** https://tryhackme.com/room/becomeadefender
**Format:** Reading + Defensive Scenario Exercise

### What It Covers

Defenders (the Blue Team) protect the same systems that attackers target. Having worked through the hacker's perspective in the previous room, this room looks at the same landscape from the other side. Understanding how attacks work is a prerequisite for defending against them effectively.

**Core Defensive Concepts:**

```
+----------------------------------------------------+
|  DEFENSIVE SECURITY FRAMEWORK                     |
|                                                   |
|  THREAT          A potential danger (hacker,      |
|                  malware, insider)                |
|                                                   |
|  RISK            Likelihood × Impact of a         |
|                  threat successfully occurring    |
|                                                   |
|  PREVENTION      Stop threats before they cause   |
|                  harm (firewalls, patching, MFA)  |
|                                                   |
|  DETECTION       Identify threats or suspicious   |
|                  activity (SIEM, IDS, logging)    |
|                                                   |
|  MITIGATION      Reduce or stop the impact once   |
|                  a threat is identified (isolate, |
|                  block, recover)                  |
+----------------------------------------------------+
```

**Layers of Defence:**

No single security control stops all attacks. Defence in depth means layering multiple controls so that if one fails, others remain.

```
Layer 1:  Perimeter (firewall, IPS, email filtering)
Layer 2:  Endpoint (AV, EDR, patching, hardening)
Layer 3:  Identity (MFA, strong passwords, PAM)
Layer 4:  Data (encryption, DLP, backups)
Layer 5:  Detection (SIEM, logging, threat hunting)
Layer 6:  Response (IR plan, playbooks, forensics)
```

**The Blue Team Toolkit:**

| Tool Category | Purpose | Examples |
|---|---|---|
| SIEM | Centralise and correlate logs | Splunk, Microsoft Sentinel, IBM QRadar |
| IDS/IPS | Detect/block network intrusions | Snort, Suricata, Zeek |
| EDR | Endpoint detection and response | CrowdStrike, SentinelOne, Defender for Endpoint |
| Firewall | Control network traffic | pfSense, Palo Alto, Cisco ASA |
| Vulnerability Scanner | Find weaknesses before attackers do | Nessus, Qualys, OpenVAS |
| Threat Intelligence | Know what attackers are doing | MISP, OSINT feeds, VirusTotal |

**Connecting the Two Perspectives:**

The Become a Hacker room used Gobuster to find `/login` and then brute forced weak credentials. A defender would prevent this by:

- Rate limiting login attempts (blocks automated brute force)
- Locking accounts after N failed attempts
- Removing or requiring VPN access for admin panels
- Enforcing strong passwords or MFA on admin accounts
- Monitoring for multiple failed logins (SIEM alert)

Understanding the attack is what tells you exactly which defensive controls are relevant.

**Task Answers**

| Task | Question | Answer |
|---|---|---|
| Task 1 | No answer needed | - |
| Task 2 | What is a potential danger that could harm systems or data called? | Threat |
| Task 2 | What is the process of stopping threats before they cause harm? | Prevention |
| Task 2 | What is the process of identifying threats or suspicious activity? | Detection |
| Task 2 | What actions reduce or stop the impact of a threat once identified? | Mitigation |
| Task 2 | What is the likelihood and potential impact of a threat called? | Risk |
| Task 3 | Complete the room | No answer needed |

---

## What I Learned / Reinforced

**The CIA Triad is the lens through which every security decision should be evaluated.** When I was monitoring SIEM alerts at Teleperformance, incidents were sometimes described in vague terms: "something suspicious happened". Framing every incident through CIA immediately sharpens the analysis: was data exposed (confidentiality), was it modified (integrity), or was a service disrupted (availability)? That framing tells you what the attacker's goal was and what the actual business impact is.

**Cryptography underpins almost everything in security.** HTTPS, VPNs, SSH, signed software packages, password hashing, certificate-based authentication: all of these are cryptography. My MSc dissertation on Monero forensics involved understanding ring signatures, stealth addresses and Pedersen commitments, all of which are cryptographic constructs. The Cryptography Concepts room is a beginner introduction but the underlying principles, confidentiality through encryption and integrity through hashing, run all the way up to advanced blockchain cryptography.

**`THM{born_to_hack!}` was a small thing but the methodology behind it matters.** Directory enumeration and credential brute force are two of the most common initial access techniques documented in real breach investigations. Running Gobuster against a web target is something I would do on any engagement. The fact that `qwerty` worked on an admin account is exactly the kind of finding that appears in pentest reports. These are not contrived exercises.

**The defender perspective requires knowing the attacker perspective.** The best SOC analysts and incident responders I have come across do not just know security products: they understand how attacks work at a technical level. Knowing that an attacker would use Gobuster to enumerate directories tells you what logs to look for (large numbers of 404s from one IP, user-agent strings matching scanning tools). The Pre-Security path deliberately ends with both perspectives for this reason.

**This path is complete. The real work starts now.** Pre-Security built the foundation. Cybersecurity 101 and the SOC Level 1 path are where the practical defender skills develop: log analysis, SIEM queries, malware analysis, network forensics. The goal is to connect these rooms to real SOC work and document that connection in every writeup going forward.

---

## Pre-Security Path Complete

```
+--------------------------------------------------+
|  PRE-SECURITY PATH - COMPLETED                  |
|                                                 |
|  Module 1  - Intro to Cyber Security      DONE |
|  Module 2  - Network Fundamentals         DONE |
|  Module 3  - How The Web Works            DONE |
|  Module 4  - Computer Fundamentals        DONE |
|  Module 5  - Operating Systems Basics     DONE |
|  Module 6  - Software Basics              DONE |
|  Module 7  - Attacks and Defenses         DONE |
|                                                 |
|  Next path: Cybersecurity 101                  |
+--------------------------------------------------+
```

---

## Resources

- [The CIA Triad - TryHackMe](https://tryhackme.com/room/theciatriad)
- [Cryptography Concepts - TryHackMe](https://tryhackme.com/room/cryptographyconcepts)
- [Become a Hacker - TryHackMe](https://tryhackme.com/room/becomeahacker)
- [Become a Defender - TryHackMe](https://tryhackme.com/room/becomeadefender)
- [Verizon DBIR 2024 - Data Breach Investigations Report](https://www.verizon.com/business/resources/reports/dbir/)
- [MITRE ATT&CK Framework](https://attack.mitre.org/)
- [OWASP Top 10](https://owasp.org/www-project-top-ten/)
- [Gobuster GitHub](https://github.com/OJ/gobuster)

---

*Written by fiza.sk293 · [GitHub](https://github.com/FizaShaikh293/thm-writeups)*
