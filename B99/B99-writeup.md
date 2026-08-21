# [Brooklyn Nine Nine](https://tryhackme.com/room/brooklynninenine)

| Field      | Details                          |
|------------|----------------------------------|
| Room       | [Brooklyn Nine Nine](https://tryhackme.com/room/brooklynninenine) |
| Platform   | TryHackMe                        |
| Difficulty | Beginner                          |
| Category   | Linux, Privilege Escalation        |
| Tags       | FTP Enumeration, SSH Brute-Force, Steganography, GTFOBins |

---

## Overview

This room is aimed for beginner level hackers but anyone can try to hack this box. There are two main intended ways to root the box, and both start from the same three open services. Nothing here needs a fancy exploit chain: it's a straightforward exercise in enumerating what's exposed, noticing a weak credential somewhere in that exposure, and turning a low-privilege shell into root through a sudo misconfiguration.

The box only exposes FTP, SSH, and HTTP. Every path to root runs back through one of those three ports, which makes it a solid room for practicing the "enumerate everything before you touch anything" habit.

```
Attack surface
┌─────────────────────────────────────┐
│  21/tcp – FTP (anonymous login)       │
│  22/tcp – SSH                          │
│  80/tcp – HTTP (single image + note)   │
└─────────────────────────────────────┘
```

---

## Enumeration

### Initial Scan

A standard service/version scan against the target lays out the whole attack surface immediately:

```bash
nmap -sC -sV -Pn <TARGET_IP>
```

```
PORT   STATE SERVICE
21/tcp open  ftp
22/tcp open  ssh
80/tcp open  http
```

Three ports, no red herrings. The room's difficulty comes from patience with enumeration, not from complexity in what's running.

### Port 80 — A Single Image and a Hint

The web root serves a single Brooklyn 99–themed image and nothing else visible on the page itself. Viewing the page source turns up a comment directly pointing toward steganography as one of the two intended routes, which is the first branch in the room.

### Port 21 — Anonymous FTP

The FTP service accepts an `anonymous` login with a blank password, which is the second branch:

```bash
ftp <TARGET_IP>
# Name: anonymous
# Password: (blank, just press enter)
```

Listing the directory turns up a file named `note_to_jake.txt`. Downloading and reading it (`get note_to_jake.txt`, then `cat` locally) reveals a short message, apparently from a colleague to a user named **Jake**, hinting that his password isn't a strong one.

```
FTP enumeration chain
┌───────────────────────────────────────────┐
│  Anonymous FTP login                        │
│         │                                   │
│  note_to_jake.txt ──► username: jake         │
│                      ──► hint: weak password │
└───────────────────────────────────────────┘
```

That single file gives us everything needed for the more direct of the two root paths: a valid username and a strong signal that it's crackable.

---

## Initial Access

### Path A — Brute-Forcing SSH

With a candidate username in hand, the note's "weak password" hint points straight at an SSH brute-force rather than anything on the web or FTP services:

```bash
hydra -l jake -P /usr/share/wordlists/rockyou.txt ssh://<TARGET_IP>
```

Hydra returns a working password quickly, since it's low down a common wordlist. Logging in confirms it:

```bash
ssh jake@<TARGET_IP>
```

This drops straight into a shell as `jake`, where the user flag is sitting in the home directory.

```
Answer 1 — User flag: ee11cbb19052e40b07aac0ca060c23ee
```

### Path B — Steganography in the Web Image

The alternative, slower route runs entirely through port 80. Downloading the image served on the site and checking it for hidden data with `steghide` requires a password:

```bash
steghide extract -sf brooklyn99.jpg
```

Since the password isn't known upfront, `stegcracker` brute-forces it against the same image using a standard wordlist:

```bash
stegcracker brooklyn99.jpg /usr/share/wordlists/rockyou.txt
```

Once cracked, the recovered password can be fed back into `steghide` to pull the hidden data out of the image, which yields a set of SSH credentials. Either path — Hydra against SSH or stego-cracking the image — lands in the same place: a shell as a low-privileged user.

```
Two paths, one outcome
┌───────────────────────────────────────────────┐
│  Path A: note hint ──► Hydra ──► SSH as jake     │
│  Path B: image ──► stegcracker ──► steghide ──►  │
│           creds ──► SSH                          │
│         │                                        │
│  Both converge: low-priv shell + user flag        │
└───────────────────────────────────────────────┘
```

---

## Escalating Privileges

### Checking Sudo Rights

Once inside as `jake`, a quick check of what the account is allowed to run as another user reveals the escalation path immediately:

```bash
sudo -l
```

The account has passwordless sudo rights to run `less` — a program never intended to be a privilege-escalation vector, but one that GTFOBins documents extensively for exactly this scenario.

### Abusing `less` via GTFOBins

`less` can spawn a shell from within its pager interface using `!`, and if it's invoked through `sudo`, that shell inherits root privileges:

```bash
sudo less /etc/hosts
# inside the pager:
!/bin/sh
```

This drops into a root shell, where the root flag is located.

```
Answer 2 — Root flag: 63a9f0ea7bb98050796b649e85481845
```

```
Privilege escalation chain
┌────────────────────────────────────────────┐
│  sudo -l ──► jake can run `less` as root      │
│         │                                     │
│  GTFOBins `less` entry ──► !/bin/sh escape     │
│         │                                     │
│  Root shell ──► root flag                      │
└────────────────────────────────────────────┘
```

---

## Investigation Summary

```
[nmap scan]
  │
  └─ 21/ftp, 22/ssh, 80/http
         │
[Port 80]                          [Port 21]
  │                                   │
  Image + source comment           Anonymous login
  │                                   │
  Steganography hint            note_to_jake.txt
  │                                   │
  stegcracker + steghide          username: jake
  │                                   │
  Recovered SSH creds ◄──────► Hydra brute-force
         │
[SSH as jake] ──► user flag
         │
  sudo -l ──► less (GTFOBins)
         │
[Root shell] ──► root flag
```

---

## Flags

| # | Question    | Answer                             |
|---|-------------|--------------------------------------|
| 1 | User flag   | `ee11cbb19052e40b07aac0ca060c23ee`   |
| 2 | Root flag   | `63a9f0ea7bb98050796b649e85481845`   |

**Would I recommend this room?** Yes — it's a clean, low-frustration introduction to combining basic enumeration (FTP, HTTP source) with credential attacks (Hydra, stegcracker) and a textbook GTFOBins privilege escalation. Good first "real" box for anyone past the absolute fundamentals.

---

## Blue Team Takeaways

**Anonymous FTP Is Still a Live Foot-Gun**
Anonymous FTP access handed over a file that named a real user and hinted at a weak password, no exploitation required. Anonymous read access to any file share should be treated as a public disclosure surface: assume everything in it will be read by someone hostile, and never leave operational notes, usernames, or hints about credentials sitting in it.

**Weak Passwords Are Still the Fastest Way In**
Both intended paths to a foothold ended at the same place: a crackable password, whether guessed via Hydra against SSH or hidden behind a weak steghide passphrase. Wordlist attacks against common wordlists like rockyou.txt succeed constantly in the real world for the same reason they succeed in this room — people reuse short, guessable passwords. Enforcing password complexity and rate-limiting or lockout on SSH would have closed this path entirely.

**Passwordless Sudo Access Needs a Reason to Exist**
Letting a low-privileged account run any binary as root without a password is a privilege escalation waiting to happen, and GTFOBins exists specifically to catalogue how many "harmless" utilities (`less`, `vim`, `find`, `awk`, and dozens more) can spawn a shell. Any `sudoers` entry should be scoped as tightly as possible, and audited against GTFOBins before being shipped.

**Hidden Data in Images Is Only as Strong as Its Password**
Steganography hid the second set of credentials, but the protection was only as good as the steghide passphrase, which folded to the same rockyou.txt wordlist used elsewhere in the room. Hiding data is not the same as protecting it; whatever secures the hidden payload needs the same password strength as anything exposed in the clear.

---

*Two roads, one root shell — Brooklyn Nine Nine proves you don't need a zero-day when a note file and a common wordlist will do the job just fine.*
