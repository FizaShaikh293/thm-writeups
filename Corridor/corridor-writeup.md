# [Corridor](https://tryhackme.com/room/corridor)

| Field      | Details                          |
|------------|----------------------------------|
| Room       | [Corridor](https://tryhackme.com/room/corridor) |
| Platform   | TryHackMe                        |
| Difficulty | Easy                                |
| Category   | Web Exploitation                    |
| Tags       | IDOR, MD5 Obfuscation, Burp Suite Intruder |

---

## Overview

*"Can you escape the Corridor?"* Sure, mostly because the Corridor's idea of a lock is thirteen doors labeled with MD5 hashes and a straight face daring you not to notice. This is the web security equivalent of hiding your house key under the doormat, except the doormat is also labeled "KEY UNDERNEATH" in hex.

The whole room is a single, elegant lesson in Insecure Direct Object Reference: take something that should be an access control decision, hash it with an algorithm everyone's rainbow table has already cracked a billion times over, and call it a day. Thirteen visible doors, thirteen suspiciously hash-shaped URLs, and one door that was never meant to be seen, hiding in plain sight at the one index number nobody bothered to list on the sign.

```
Attack surface
┌─────────────────────────────────────┐
│  80/tcp – HTTP (Flask/Werkzeug app,    │
│           thirteen doors, one corridor)│
└─────────────────────────────────────┘
```

---

## Enumeration

### Initial Scan

```bash
nmap -sV -sC <TARGET_IP>
```

```
PORT   STATE SERVICE VERSION
80/tcp open  http    Werkzeug httpd (Python)
```

One port, one page, one corridor. This room isn't hiding behind a firewall or a fancy WAF, it's hiding behind the assumption that nobody reads page source. Bold assumption for a website built entirely out of doors.

### Walking the Corridor

Loading the site up drops you into a hallway lined with doors, thirteen of them, each one clickable, each one leading to a different, entirely empty room. Clicking through manually is a great way to feel like you're getting somewhere while actually going nowhere, so naturally that's not the move.

Hovering over a door, or just looking at the URL after clicking one, gives the game away immediately:

```
http://<TARGET_IP>/c4ca4238a0b923820dcc509a6f75849b
```

That's not a session token. That's not a nonce. That's an MD5 hash wearing a trench coat, and it's a bad disguise, because MD5 hashes of small integers have been sitting in every rainbow table on the internet since roughly the dawn of rainbow tables.

```
IDOR setup
┌───────────────────────────────────────────┐
│  Corridor homepage ──► 13 clickable doors     │
│         │                                     │
│  Each door's URL ──► MD5-looking hex string     │
│         │                                     │
│  Hover / click ──► confirms it's the room ID     │
└───────────────────────────────────────────┘
```

### Speedrunning the Hash List

Manually clicking all thirteen doors and copying hashes one at a time is technically a strategy, if you enjoy suffering. Viewing the page source instead hands over every door's hash in one go, all thirteen `href` values sitting right there in the HTML, no clicking required.

Running each hash through a lookup tool like CrackStation confirms the obvious: they're MD5 digests of the numbers **1 through 13**, in order, matching the doors on screen exactly. Somebody built access control out of `md5(str(door_number))` and called it obfuscation.

---

## Exploiting the IDOR

### The Math That Matters

Thirteen doors, numbered 1 to 13, in a room built around the premise that URL endpoints deserve a second look. If the visible doors go from 1 to 13, the actually interesting question isn't "what's door 14," it's "what happened to door 0." Programmers count from zero. Doors don't, apparently, but servers still do.

```bash
echo -n "0" | md5sum
# cc700ea9f7f81aa1eeb5478cb0e7b0aa (example — hash the string, not the room)
```

Hashing the string `"0"` and dropping the resulting MD5 straight into the URL bar in place of one of the visible door hashes reaches an endpoint that was never linked anywhere on the page, never shown as a door, and never meant to be found by anyone clicking around the UI like a normal, well-behaved website visitor.

```
IDOR exploitation chain
┌───────────────────────────────────────────┐
│  Doors visible ──► numbered 1 through 13       │
│         │                                     │
│  Hypothesis: room 0 exists, just not linked     │
│         │                                     │
│  md5("0") ──► crafted URL                        │
│         │                                     │
│  Navigate directly ──► hidden room 0              │
└───────────────────────────────────────────┘
```

### Confirming It With Burp (Optional, But Satisfying)

For anyone who wants to watch the whole hallway get brute-forced instead of guessing one door at a time, Burp Suite's Intruder makes short work of it: capture a request to any door with intercept on, send it to Intruder, mark the hash portion of the URL as the payload position, and feed it a payload list of MD5 hashes for a range of integers (`seq 0 25 | md5sum` territory) to sweep every plausible room number in one pass.

```
Burp Intruder sweep
┌───────────────────────────────────────────┐
│  Capture GET /<hash> ──► send to Intruder       │
│         │                                     │
│  Mark hash as payload position                    │
│         │                                     │
│  Payload list: MD5(0) .. MD5(25)                   │
│         │                                     │
│  Response lengths flag the odd one out ──► room 0  │
└───────────────────────────────────────────┘
```

Either method, hand-crafted single hash or automated sweep, lands on the same door: room 0, sitting quietly outside the numbered range the UI ever admits to having, holding the flag the whole time.

```
Answer 1 — Flag: flag{2477ef02448ad9156661ac40a6b8862e}
```

---

## Investigation Summary

```
[nmap scan]
  │
  └─ 80/http only
         │
[Corridor homepage] ──► 13 clickable doors
         │
[Page source] ──► 13 MD5-looking href hashes
         │
[Hash lookup] ──► hashes = MD5(1) through MD5(13)
         │
[Pattern recognized] ──► UI only shows 1–13, servers count from 0
         │
[md5("0") crafted] ──► navigate directly / Burp Intruder sweep
         │
[Hidden room 0] ──► flag
```

---

## Flags

| # | Question | Answer |
|---|----------|--------|
| 1 | What is the flag? | `flag{2477ef02448ad9156661ac40a6b8862e}` |

**Would I recommend this room?** Yes, especially as a first IDOR room. It's a tight, single-concept challenge: no rabbit holes, no privilege escalation, just one clean lesson in why hashing a predictable value is not the same thing as access control. Ten minutes well spent for anyone still fuzzy on what IDOR actually looks like in practice.

---

## Blue Team Takeaways

**Hashing Is Not Access Control**
MD5-ing a room number doesn't make it unguessable, it just makes it look unguessable to anyone who doesn't bother checking. Any value used to gate access to a resource needs an actual authorization check behind it, tied to who's asking, not just what string they happen to be holding.

**"Not Linked" Is Not the Same as "Not Reachable"**
Room 0 was never shown in the UI, and that was the entirety of its protection. Hidden-but-reachable endpoints are exactly as exposed as visible ones to anyone willing to enumerate, and "we didn't put a link to it" has never once stopped a determined visitor with a hash calculator.

**Predictable Identifiers Are a Gift to Attackers**
Once the pattern (MD5 of a sequential integer) was spotted on door 1, every other door, visible or hidden, fell out of that same pattern instantly. Sequential or otherwise-predictable IDs, hashed or not, should never be trusted to double as a security boundary; use random, unguessable identifiers (or better, real authorization checks) for anything that shouldn't be freely enumerable.

**Client-Facing Source Code Reveals Server-Side Structure**
Viewing the page source handed over all thirteen hashes in one shot, no brute-forcing required to find the pattern. Anything shipped to the browser, including the full list of "valid" resource identifiers, should be assumed fully readable by the person on the other end of the connection.

---

*Thirteen doors, twelve dead ends, and one door numbered zero that nobody thought to mention — the Corridor's real trick wasn't hiding the exit, it just never put it on the map.*
