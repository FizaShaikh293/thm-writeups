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

Somewhere in the Nine-Nine precinct, someone set up a box, slapped a Jake Peralta–core image on port 80, hid a password inside it like it was a evidence locker, and then left a note to Jake lying around on an anonymous FTP share like a Post-it on a break room fridge. Bold strategy for a precinct full of detectives.

The room bills itself as beginner-friendly, and it is, but "beginner-friendly" here really means "beginner-friendly if your beginner instincts are to check literally everything before touching anything." There are two intended ways to root the box, and neither of them requires an exploit so much as it requires noticing that somebody, somewhere, picked a password a rockyou.txt run would find in about the time it takes to say "cool cool cool cool cool, no doubt no doubt no doubt."

The box only exposes FTP, SSH, and HTTP, three ports doing the work of a whole cast of overconfident detectives, and every route to root loops back through one of them.

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

Three ports, no red herrings, no plot twist. This box isn't hiding a fourth service behind a knock sequence; it's just waiting to see if you'll actually look at what's in front of you.

### Port 80 — A Single Image and a Hint

The web root serves up exactly one Brooklyn 99–themed image and absolutely nothing else, which is either minimalist web design or a budget of zero dollars, hard to say. Viewing the page source, because clicking around a static image gets old fast, turns up a comment practically begging you to try steganography. Subtle as Charles Boyle at a crime scene.

### Port 21 — Anonymous FTP

The FTP service happily accepts an `anonymous` login with a blank password, security posture of a precinct break room:

```bash
ftp <TARGET_IP>
# Name: anonymous
# Password: (blank, just press enter)
```

Listing the directory turns up a file called `note_to_jake.txt`, which is exactly as juicy as it sounds. Downloading and reading it (`get note_to_jake.txt`, then `cat` locally) reveals a short message from a colleague to a user named **Jake**, gently roasting him for having a weak password. Somewhere, Amy Santiago is shaking her head.

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

With a name and a public roast in hand, the note practically hands you the attack plan: go brute-force the guy's SSH password.

```bash
hydra -l jake -P /usr/share/wordlists/rockyou.txt ssh://<TARGET_IP>
```

Hydra finds it embarrassingly fast, because it's parked near the top of a wordlist that's cracked more accounts than Jake's cracked jokes. Logging in confirms it:

```bash
ssh jake@<TARGET_IP>
```

Straight into a shell as `jake`, no drama, no negotiation, no hostage situation. The user flag is waiting in the home directory like it was expecting company.

```
Answer 1 — User flag: ee11cbb19052e40b07aac0ca060c23ee
```

### Path B — Steganography in the Web Image

For anyone who prefers the scenic route, path B runs entirely through port 80. Downloading the image and pointing `steghide` at it demands a password up front, because apparently even hidden data in this room can't resist a little gatekeeping:

```bash
steghide extract -sf brooklyn99.jpg
```

Since nobody handed us that password, `stegcracker` does the honors, brute-forcing it against the same tired wordlist:

```bash
stegcracker brooklyn99.jpg /usr/share/wordlists/rockyou.txt
```

Once it cracks, feeding that password back into `steghide` pulls a set of SSH credentials straight out of a picture of a precinct-branded image, which is either clever or a cry for help depending on how you look at it. Either path, brute Hydra or stego-crack an image, lands you in the exact same place: a shell as a low-privileged user who really should've picked a better password.

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

Once inside as `jake`, checking what he's allowed to run as someone else answers the "how do we root this" question almost instantly:

```bash
sudo -l
```

Jake has passwordless sudo rights to run `less`, the humble pager, the last program anyone should trust with root. GTFOBins has known about this one for years, and this box seems determined to prove the point.

### Abusing `less` via GTFOBins

`less` can spawn a shell from inside its own pager view with a single `!`, and when `less` itself was launched via `sudo`, that shell inherits root without so much as a background check:

```bash
sudo less /etc/hosts
# inside the pager:
!/bin/sh
```

And just like that, root. No exploit, no CVE, just a sudoers entry somebody really shouldn't have written. The root flag is sitting right there, unbothered.

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

**Would I recommend this room?** Cool, cool, cool, cool, cool — yes, no doubt. It's a clean, low-frustration intro to enumeration (FTP, HTTP source), credential attacks (Hydra, stegcracker), and a textbook GTFOBins privesc, all wrapped in a Brooklyn 99 bow. Good first "real" box for anyone past the absolute fundamentals, and a fine reminder that most "hacking" is just reading the note somebody left lying around.

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
