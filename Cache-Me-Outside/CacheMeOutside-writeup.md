# [Cache Me Outside](https://tryhackme.com/room/cachemeoutside)

| Field      | Details                          |
|------------|----------------------------------|
| Room       | [Cache Me Outside](https://tryhackme.com/room/cachemeoutside) |
| Platform   | TryHackMe                        |
| Difficulty | Medium                            |
| Category   | OSINT, Active OSINT               |
| Tags       | Username Correlation, Git Metadata Leakage, Email OSINT, Image Geolocation |

---

## Overview

Years after walking away from the scene, a retired hacker rebuilt his whole life around trail running and the outdoors. Character development, allegedly. Except he did it the same way he probably used to write code: fast, a little sloppy, and with his old habits leaking through every new profile he made. Our job is to reconstruct his full identity, real name, email, phone number, city, and even a specific tram stop on a specific day, using nothing but what he's put on the public internet himself.

No exploitation, no malware, no brute force. From a blue team perspective this room is really an OPSEC failure case study told from the attacker's chair. Every answer comes from a small, individually-forgivable mistake: a Git config that never got scrubbed, an out-of-office auto-reply nobody thinks twice about turning on, a billboard caught in the background of a photo. None of these are vulnerabilities in the traditional sense. They're the digital equivalent of leaving your name tag on after you clock out.

---

## Enumeration

### The Starting Point

The room hands over one asset: a leaked screenshot of a conversation between two people, one of whom drops a link to a fitness-tracking profile before the chat cuts off.

```
Investigation entry point
┌─────────────────────────────────────────┐
│  Leaked conversation screenshot           │
│         │                                │
│  Link shared mid-chat ──► Komoot profile │
└─────────────────────────────────────────┘
```

### Pivoting Off the First Profile

The Komoot profile (`komoot.com/user/5667624959835`) is the whole investigation's anchor point. Fitness-tracking platforms like Komoot are a goldmine for this kind of work precisely because people treat them as low-stakes, unlike a "real" social account, so they tend to write honest bios and link out to other accounts without thinking about it.

The profile displays a full name and a bio describing exactly the premise of the room: an ex-hacker trying to go straight through running and the outdoors. It also links out to a GitHub account.

```
Answer 1 — Full name: Jim Lee
```

```
First pivot chain
┌───────────────────────────────────────┐
│  Komoot profile ──► name: Jim Lee       │
│         │           bio: ex-hacker,     │
│         │           outdoor/running     │
│         └──► linked GitHub: jiml33t     │
└───────────────────────────────────────┘
```

---

## Initial Access

### Leaking the Email Through Git Metadata

The room's name is the biggest hint in the whole challenge: *cache* me outside. GitHub caches far more than the rendered page shows. Every commit has an author name and email baked into it from the committer's local Git config, and that data stays retrievable through a repo's `.patch` files even when the visible repo content is scrubbed clean.

```bash
# get the SHA of the first commit on the profile repo
curl -s "https://api.github.com/repos/jiml33t/jiml33t/commits?per_page=1"

# pull the raw patch for that commit
curl -sL "https://github.com/jiml33t/jiml33t/commit/<SHA>.patch"
```

```
From: jimleepro1-cell <jimleepro1@gmail.com>
Date: Thu, 16 Apr 2026 03:27:19 -0400
Subject: [PATCH] Initial commit
```

The patch reveals both the leaked email and an older username, `jimleepro1-cell`, that never got the profile cleanup the newer `jiml33t` account did.

```
Answer 2 — Email: jimleepro1@gmail.com
```

```
Metadata leak chain
┌───────────────────────────────────────────┐
│  GitHub profile repo ──► looks clean        │
│         │                                   │
│  .patch file on first commit                │
│         │                                   │
│  Git config author/email ──► leaked email   │
│                            ──► older username│
└───────────────────────────────────────────┘
```

---

## Escalating the Investigation

### The Phone Number, the Hard Way and the Easy Way

The Komoot profile also hides a password-gated "Secret Menu," which reads as a direct wink at the room's name and feels like it should be the answer. It's a red herring. The actual technique is far less flashy: send an email to the address that just leaked and see what bounces back.

Jim has an out-of-office auto-reply configured on that address, and auto-replies are written to be maximally informative because they're meant for legitimate clients, not investigators. His includes a direct contact number.

```
Answer 3 — Phone: +40 743 321 239
```

The reply also contains a small `0x4A4C` hex string tucked in as a signature touch, which decodes to "JL". Old hacker habits die hard even in an away message. The `+40` country code also lines up with everything found afterward: Romania.

```
Auto-reply OSINT chain
┌────────────────────────────────────────┐
│  Leaked email ──► send test email        │
│         │                                │
│  Out-of-office auto-reply                │
│         ├─► phone number                 │
│         └─► 0x4A4C easter egg (= "JL")   │
└────────────────────────────────────────┘
```

### Finding the City

Working the `jiml33t` username across platforms turns up an Instagram and a Threads account. Instagram is mostly empty, but the Threads account includes a photo from a run: a road shot with a billboard reading **IRIGAȚII.RO** clearly visible in the background.

That company name is specific enough to search directly. Irigații.ro is a Romanian irrigation supplier with a real, documented address on Calea Stan Vidrighin (formerly Calea Buziașului), described on their own site as sitting at the end of tramway line 4, in **Timișoara, Romania**.

```
Answer 4 — City: Timișoara
```

```
Image geolocation chain
┌─────────────────────────────────────────┐
│  Threads photo ──► billboard in frame     │
│         │           "IRIGAȚII.RO"         │
│  Company address lookup                   │
│         │                                 │
│  Timișoara, end of tramway line 4         │
└─────────────────────────────────────────┘
```

### Pinning the Exact Tram Station

The same Threads post carries a caption about finishing a last training run before "the big day" and grabbing coffee at "my favourite French supermarket" on the way home via tram. Each phrase narrows the location further:

- **Tramway line 4 terminus** puts him right at the Irigații.ro location already identified.
- **A French supermarket** nearby on Calea Buziașului is Auchan, a French retail chain with a store in exactly that area.
- **"Last run before the big day"** lines up with an earlier weekend-running post and the marathon-prep language in the out-of-office reply, dating the post to a specific day: 7 May 2026.

The tram stop serving both the Irigații.ro site and the nearby Auchan, at the terminus of line 4, is **Piața Gheorghe Domășneanu**.

```
Answer 5 — Tram station (7 May 2026): Piața Gheorghe Domășneanu
```

```
Final pin chain
┌────────────────────────────────────────────┐
│  Threads caption ──► tram + French supermarket│
│         │                                     │
│  Line 4 terminus + Auchan location             │
│         │                                     │
│  Piața Gheorghe Domășneanu                     │
└────────────────────────────────────────────┘
```

---

## Investigation Summary

```
[Leaked Screenshot]
  │
  └─ Komoot link ──► Jim Lee, ex-hacker bio, GitHub link
         │
[GitHub]
  │
  └─ .patch commit metadata ──► jimleepro1@gmail.com
                              ──► old username jimleepro1-cell
         │
[Email]
  │
  └─ Send test email ──► out-of-office auto-reply
                       ──► +40 743 321 239 (Romania)
         │
[Cross-Platform Pivot]
  │
  └─ jiml33t ──► Instagram (dead end) / Threads (photo + caption)
         │
[Geolocation]
  │
  ├─ Billboard "IRIGAȚII.RO" ──► Timișoara, tram line 4 terminus
  └─ Caption clues (tram, French supermarket, "big day")
                              ──► Piața Gheorghe Domășneanu
```

---

## Flags

| # | Question                                                          | Answer                          |
|---|--------------------------------------------------------------------|-----------------------------------|
| 1 | What is the retired hacker's full name?                            | `Jim Lee`                         |
| 2 | What email address did he accidentally expose?                     | `jimleepro1@gmail.com`            |
| 3 | What is his phone number?                                           | `+40 743 321 239`                 |
| 4 | In which city is he located?                                        | `Timișoara`                       |
| 5 | Tram station where he got off on 7 May 2026                        | `Piața Gheorghe Domășneanu`       |

---

## Blue Team Takeaways

**Git Metadata Outlives the Repo's Public Face**
Deleting sensitive commits or switching to a cleaner username doesn't retroactively scrub what's baked into old commit objects. `.patch` and `.diff` endpoints on GitHub serve raw commit data, author name and email included, straight from Git's own metadata regardless of how the rendered repo page looks. Anyone reusing a personal email in `git config user.email` on a public repo should assume that address is permanently public, and org-wide Git hygiene policies should enforce no-reply GitHub emails by default.

**Automated Responses Are Unsupervised Disclosure**
An out-of-office auto-reply is written to be helpful to a stranger, which is exactly the threat model an investigator fits. It handed over a direct phone number without a single human reviewing the request. Auto-responders, ticketing system replies, and bounce messages should be audited for what they disclose to an unauthenticated sender, the same way a company would audit a public-facing web form.

**A Single Photo Can Geolocate Someone Precisely**
One incidental billboard in the background of a training-run photo was enough to pin down not just a city but a specific transit terminus. Reverse image and background-object searches are trivially effective, and most people don't think to scrub backgrounds, storefronts, or signage before posting. Anyone concerned about physical location privacy needs to think about what's behind them in a photo, not just what metadata is attached to the file.

**Identity Correlation Compounds Across Unrelated Platforms**
No single account gave up the full picture. A fitness app supplied a name and a link, a code host supplied an email, an email server supplied a phone number, and a social app supplied a location. Each platform's individual exposure looked minor in isolation; chained together they produced a complete dossier. This is the same methodology real threat intel analysts use to attribute activity to a person, and it's exactly why "I didn't post anything sensitive on any one platform" is not the same thing as being anonymous.

---

*Jim went from popping shells to popping into Auchan for post-run coffee, and somehow the second one is what got him found.*
