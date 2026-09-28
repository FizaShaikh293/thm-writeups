# [Lo-Fi](https://tryhackme.com/room/lofi)

| Field      | Details                          |
|------------|----------------------------------|
| Room       | [Lo-Fi](https://tryhackme.com/room/lofi) |
| Platform   | TryHackMe                        |
| Difficulty | Easy                                |
| Category   | Web Exploitation                    |
| Tags       | Local File Inclusion, Path Traversal, Web Enumeration |

---

## Overview

*"Want to hear some lo-fi beats, to relax or study to? We've got you covered!"* Sure, and also apparently covered on "let the visitor pick which file the server reads off disk," which is a genre of exposure that vibes a lot less than the actual music. Somewhere between the chill beats and the study playlist, this box decided the `page` parameter should just trust whatever string wanders in, no ID required, no bouncer at the door.

There's no privilege escalation here, no sudo rights to abuse, no cron job quietly waiting to betray root. Just a beginner-friendly, single-concept lesson: Local File Inclusion, delivered with a soundtrack. The URL bar becomes a filesystem browser the moment you stop clicking the playlist links and start editing them yourself.

```
Attack surface
┌─────────────────────────────────────┐
│  22/tcp – SSH (present, unused)        │
│  80/tcp – HTTP (Lo-Fi music site,       │
│           page-based file inclusion)    │
└─────────────────────────────────────┘
```

---

## Enumeration

### Initial Scan

```bash
nmap -sC -sV -T4 <TARGET_IP>
```

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh
80/tcp open  http    Apache
```

SSH shows up, gets acknowledged, and then never comes up again for the rest of the room. Classic red herring energy, or more likely, just what happens when a box gets built from a template that always opens port 22 out of habit.

### Setting the Mood

Port 80 serves an honest-to-god lo-fi music site, complete with mellow track links like **Relax** and a couple of other "modes" to click through. Clicking one of them updates the URL with a `page` parameter, something along the lines of:

```
http://<TARGET_IP>/index.php?page=relax
```

Which is a very polite way of saying "tell me which file to load, I won't ask too many questions." That's not a vibe, that's a vulnerability wearing headphones.

```
Reconnaissance chain
┌───────────────────────────────────────────┐
│  Lo-Fi homepage ──► track links (Relax, etc.)  │
│         │                                     │
│  Click a track ──► URL grows a `page` param      │
│         │                                     │
│  `page` value looks suspiciously like a filename │
└───────────────────────────────────────────┘
```

---

## Initial Access

### Confirming the LFI

The room's own hint is basically a neon sign: *"find the flag in the root of the filesystem."* That's not a riddle, that's a shopping list. Testing the `page` parameter with a classic path traversal payload confirms the server will happily read whatever's put in front of it:

```
http://<TARGET_IP>/index.php?page=../../../etc/passwd
```

Back comes the full contents of `/etc/passwd`, usernames, UIDs, shells, the whole cast list, served up with zero resistance. No filtering, no whitelist of allowed pages, nothing standing between the parameter and the raw filesystem.

```
LFI confirmation chain
┌───────────────────────────────────────────┐
│  page=relax ──► loads relax.php normally       │
│         │                                     │
│  page=../../../etc/passwd ──► loads /etc/passwd  │
│         │                                     │
│  No sanitization, no whitelist ──► full LFI        │
└───────────────────────────────────────────┘
```

### Climbing to the Flag

With arbitrary file reads confirmed, the only real work left is pointing the same trick at the file the room actually cares about. The description already gave the address: root of the filesystem.

```
http://<TARGET_IP>/index.php?page=../../../../flag.txt
```

Enough `../` to clear out of the web root and back up to `/`, and `flag.txt` loads straight into the browser, no further creativity required. The site kept playing chill beats the entire time, which feels almost rude given what was happening to its filesystem in another tab.

```
Answer 1 — Flag: flag{e4478e0eab69bd642b8238765dcb7d18}
```

---

## Investigation Summary

```
[nmap scan]
  │
  └─ 22/ssh (unused), 80/http
         │
[Lo-Fi homepage] ──► track links with `page` parameter
         │
[Payload: page=../../../etc/passwd] ──► confirms LFI
         │
[Payload: page=../../../../flag.txt] ──► flag
```

---

## Flags

| # | Question | Answer |
|---|----------|--------|
| 1 | Climb the filesystem to find the flag! | `flag{e4478e0eab69bd642b8238765dcb7d18}` |

**Would I recommend this room?** Yes, especially for anyone who's never touched LFI before. It's about as gentle an introduction as the vulnerability class gets: one parameter, one payload, one flag, no distractions. Good five-minute warm-up before tackling a room where the file inclusion is buried under a few more layers of filtering.

---

## Blue Team Takeaways

**User Input Should Never Choose a File Path Directly**
The `page` parameter took a raw string and fed it straight into a file-loading function, no validation, no boundaries. Any parameter that influences which file gets read or included needs to be checked against an explicit allow-list of known-good values, never trusted as a literal path.

**Path Traversal Sequences Deserve Active Filtering**
`../` sequences walked the request straight out of the intended directory and into the rest of the filesystem without a single check catching it. Stripping or rejecting traversal sequences, and better yet, resolving the final path and confirming it still lives inside the expected directory, closes this off completely.

**"It's Just a Music Site" Is Not a Threat Model**
A low-stakes-looking front end, chill background music and all, still sat on top of a server that could be walked to `/etc/passwd` and beyond. Attack surface doesn't scale with how relaxing the branding is; every parameter that touches the filesystem needs the same scrutiny whether the app streams lo-fi beats or processes bank transfers.

**Least Privilege Limits the Blast Radius**
Even with the vulnerability present, running the web service as a low-privileged user with minimal filesystem read access would have limited how much an LFI could actually expose. Defense in depth means the application code isn't the only thing standing between a bug and a full filesystem read.

---

*Peak chill-hop hacking: no exploits fired, no shells popped, just a URL bar, a few extra `../`, and a flag that never asked to be found this easily.*
