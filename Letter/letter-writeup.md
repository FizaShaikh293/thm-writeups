# [Letter](https://tryhackme.com/room/letter)

| Field      | Details                          |
|------------|----------------------------------|
| Room       | [Letter](https://tryhackme.com/room/letter) |
| Platform   | TryHackMe                        |
| Difficulty | Easy                              |
| Category   | OSINT, Puzzle                    |
| Tags       | Open Source Intelligence, Historical Research, French Postal Codes, Image Analysis |

---

## Overview

No nmap for this one. No shell, no privesc, no `sudo -l`. Instead of a vulnerable box you get a zip file: a water-damaged envelope, a torn newspaper clipping, and a handwritten note from someone named Audette to someone named Édouard. The task is to reconstruct a story out of fragments the same way an analyst reconstructs an incident timeline out of half a log file and a screenshot someone sent in a panic.

From a blue team perspective this room is really an OSINT tradecraft exercise wearing a period-drama costume. Every step, decoding a barcode instead of trusting a smudged handwritten digit, narrowing a date range from secondary headlines instead of the torn primary one, cross-referencing a regional archive against a dedicated historical site, is the same discipline used in real threat intel work: verify visual evidence instead of assuming it, follow corroborating details instead of the first guess, and don't stop digging when the first source runs out of detail.

---

## Enumeration

### The Evidence

The room's download gives three artifacts to work from:

```
Case files
┌───────────────────────────────────────────┐
│  envelope.png ──► torn, water damaged,     │
│                   orange barcode visible    │
│  clipping.png ──► L'Ouest-Éclair, torn      │
│                   main headline, intact     │
│                   secondary headlines       │
│  note.txt     ──► handwritten letter,       │
│                   French, addressed to      │
│                   "Édouard"                 │
└───────────────────────────────────────────┘
```

The envelope is addressed to **Édouard G.**, care of the **SNSM** (Société Nationale de Sauvetage en Mer), France's sea rescue organisation. The note is signed by **Audette** and references "your great-grandfather," so the chain of custody here is: old newspaper clipping found in an attic, tucked into a letter, sent to a descendant of whoever the clipping is actually about.

The note itself, translated, reads:

> "My dear Édouard, today, while tidying the attic at my grandparents' house, I came across this old newspaper clipping. Your great-grandfather wasn't even old enough to get his driver's license when he distinguished himself that day. The youngest of the team, and certainly not the least courageous. He would be so proud to see you on the water too. With all my love, Audette."

Two hard facts drop out of that paragraph immediately: the person we're after was the **youngest member of a rescue team**, and he was **under driving age**, which in 1920s France means under 18.

```
What the note actually tells us
┌────────────────────────────────────┐
│  "youngest of the team"  ──► age rank│
│  "not old enough for driver's license"│
│                        ──► under 18   │
└────────────────────────────────────┘
```

---

## Initial Access

### Decoding the Postal Code

Task one is the postal code on the envelope. The handwritten digits are smeared by water damage into something that could plausibly be read as `53432`, and that reading is a trap: department `53` is Mayenne, landlocked and nowhere near the sea. No sea rescue station has any business being there. That mismatch alone is the signal to stop trusting the handwriting.

The envelope has a second, more reliable data source sitting in plain sight: a row of orange bars, a French postal barcode used by sorting machines to encode the address digit by digit, independent of anyone's handwriting.

```
..||||| |.||.| ||..|| |||..| .||.||
```

Each 6-bar group maps to a digit using the standard French postal barcode chart:

```
0 ..||||   3 .|||.|   6 |.||.|   9 |||..|
1 .|.|||   4 |..|||   7 ||..||
2 .||.||   5 |.|.||   8 ||.|.|
```

Decoding group by group (the first bar is a start marker, ignored):

```
|.||.|  → 6
||..||  → 7
|||..|  → 9
.||.||  → 2
```

That gives `6792`. French postal barcodes read right to left, so flipped it becomes:

```
Postal code: 29760
```

29760 is Penmarc'h, a commune on the Finistère coast, exactly where an SNSM sea rescue station would be. The barcode agreed with geography where the handwriting didn't.

```
Postal code chain
┌───────────────────────────────────────────┐
│  Smeared handwriting ──► looks like 53432   │
│         │            (landlocked, wrong)   │
│  Orange barcode ──► decoded digit by digit  │
│         │                                   │
│  Reversed (right-to-left) ──► 29760         │
│         │                                   │
│  Geography check ──► Penmarc'h, coastal ✓   │
└───────────────────────────────────────────┘
```

---

## Escalating the Investigation

### Dating the Newspaper Clipping

The clipping's main headline, the one that would name the actual disaster, is torn away. But two intact secondary headlines are enough to bracket a date:

- *"Amundsen a-t-il atteint le pôle Nord?"* (Has Amundsen reached the North Pole?)
- A line referencing Painlevé and Herriot, French political figures active together in a narrow window in 1925

Roald Amundsen's Arctic flight attempt, and the "has he been lost" coverage that followed, along with Paul Painlevé's term as Prime Minister starting in April 1925, both point at the same narrow window: **late May 1925**. Searching regional archive scans of *L'Ouest-Éclair* around that date confirms the exact issue date, and while browsing that range a second, more specific article turns up describing a storm disaster off the Finistère coast the day before.

```
Date-narrowing chain
┌───────────────────────────────────────────┐
│  Torn headline ──► useless directly         │
│  Secondary headlines ──► Amundsen story +   │
│                          Painlevé/Herriot   │
│         │                                   │
│  Cross-reference historical dates           │
│         │                                   │
│  Window: late May 1925                       │
│         │                                   │
│  Archive lookup ──► exact issue confirmed    │
│                     + adjacent disaster story│
└───────────────────────────────────────────┘
```

### Finding the Disaster

The torn main headline is legible enough in fragments to read as a catastrophe in Finistère involving two boats and two rescue lifeboats, with drownings. Combined with the confirmed date, this points squarely at the **Penmarc'h lifeboat disaster of 23 May 1925**: a sudden storm capsized rescue boats attempting to save a fishing crew, and multiple men died. This is a real, documented historical event, not a fabricated CTF backstory, which is part of what makes the research chain in this room genuinely work like OSINT rather than puzzle-solving.

### Identifying the Rescuer

General disaster confirmed, now for the specific name. Regional archive detail on casualty lists runs out fast, so the next move is a dedicated local historical resource rather than a national newspaper: a site maintained around the history of Penmarc'h itself, with a section specifically archiving the 23 May 1925 disaster and its crew roster.

Cross-referencing that roster against what the note told us, youngest of the team, too young to drive, turns up one name that fits both conditions at once:

```
Identity confirmation chain
┌────────────────────────────────────────────┐
│  Note: youngest crew member, under 18       │
│         │                                    │
│  Regional archive: crew roster, ages listed  │
│         │                                    │
│  GOURLAOUEN, Yves-Marie ──► age 15, mousse   │
│  (ship's boy), youngest aboard               │
│         │                                    │
│  Later awarded a Silver Medal for bravery    │
└────────────────────────────────────────────┘
```

Yves-Marie Gourlaouen, 15 years old, served as ship's boy on one of the rescue boats that day and was decorated for it afterward. That matches the note's description on both counts: youngest of the team, and years away from a driver's license.

---

## Investigation Summary

```
[Case Files]
  │
  ├─ envelope ──► SNSM, "Édouard G.", barcode
  ├─ clipping ──► L'Ouest-Éclair, torn headline
  └─ note      ──► "youngest," "under 18," great-grandfather
         │
[Postal Code]
  │
  └─ barcode decoded, reversed ──► 29760 (Penmarc'h)
         │
[Dating]
  │
  └─ Amundsen + Painlevé headlines ──► late May 1925
                                    ──► 23 May 1925 confirmed
         │
[Event]
  │
  └─ Penmarc'h lifeboat disaster, 23 May 1925
         │
[Identity]
  │
  └─ Regional archive crew roster ──► youngest, under 18
                                   ──► Yves-Marie Gourlaouen, 15
```

---

## Flags

| # | Question                                                   | Answer                              |
|---|---------------------------------------------------------------|--------------------------------------|
| 1 | What is the postal code of the delivery address on the envelope? | `29760`                              |
| 2 | Full name and age of the person in the note (flag)             | `THM{Yves-Marie_Gourlaouen_15}`      |

---

## Blue Team Takeaways

**Verify Visual Evidence Instead of Trusting the First Read**
The smeared handwriting on the envelope looked like a plausible postal code right up until it was checked against geography. In incident response the same trap shows up constantly: a timestamp, a hostname, or a log line that looks right until it's cross-checked against an independent source. Barcodes, hashes, and machine-generated data exist precisely because handwriting and manual entry are unreliable; when both are available, trust the machine-readable one and use it to sanity-check the human one.

**Corroborate With Independent, Secondary Details**
The primary headline, the one piece of evidence that would have made this trivial, was destroyed. The investigation only moved because two unrelated secondary headlines both independently pointed at the same narrow date window. Real investigations rarely get the one perfect log line either; triangulating a timeline from several partial, independent sources is a core skill, not a workaround.

**Know When to Escalate to a More Specific Source**
National and regional archives confirmed the event but ran out of detail on individual crew members. The answer required stepping down to a niche, purpose-built local historical resource. The same escalation applies in threat intel: broad feeds confirm that something happened, but attribution-level detail usually lives in a narrower, more specialized source, and knowing that one exists (and where to look) is what separates a surface-level lookup from a real investigation.

---

*Some cold cases are 100 years old, wet, and torn in half, and you still find the kid.*
