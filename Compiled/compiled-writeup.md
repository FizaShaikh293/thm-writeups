# [Compiled](https://tryhackme.com/room/compiled)

| Field      | Details                          |
|------------|----------------------------------|
| Room       | [Compiled](https://tryhackme.com/room/compiled) |
| Platform   | TryHackMe                        |
| Difficulty | Easy                                |
| Category   | Reverse Engineering                 |
| Tags       | Static Analysis, Ghidra, strcmp Logic, Format String Parsing |

---

## Overview

*"Strings can only help you so far."* Which is the room's way of saying: yes, you can absolutely run `strings` on this binary and feel very clever for about thirty seconds, and then you're going to have to actually open Ghidra like the rest of us. This challenge hands over one lonely ELF binary, no server, no network, no privilege escalation, just a compiled program with a password check baked into its `main()` function and a strong opinion about the format of whatever you type at it.

The whole thing hinges on a scanf format string that looks helpful right up until you realize it's quietly discarding half of what you type, and a pair of string comparisons that only look intimidating because their names came straight out of a compiler's own internals. Nobody wrote `__dso_handle` and `_init` expecting them to become a CTF password, and that's exactly why they work as one.

```
Attack surface
┌─────────────────────────────────────┐
│  One local ELF binary                 │
│  No network service, no server         │
│  Solved entirely via static analysis   │
└─────────────────────────────────────┘
```

---

## Enumeration

### Identifying the File

First things first, figure out what's actually been handed over:

```bash
file Compiled
```

```
Compiled: ELF 64-bit LSB executable, x86-64, ...
```

A standard 64-bit Linux executable, nothing exotic, nothing packed or obfuscated. Just a program that's going to ask a question and judge the answer.

### Running It Blind

Executing the binary directly confirms the obvious:

```bash
./Compiled
```

```
Password:
```

One prompt, one shot, and typing anything reasonable-sounding gets a curt `Try again!` in response. Time to stop guessing and start reading.

### The Lazy First Pass: `strings`

Running `strings` against the binary, because it's free and it's fast, turns up a suspicious-looking format string sitting right there in the printable data:

```bash
strings Compiled
```

```
Password: 
DoYouEven%sCTF
Correct!
Try again!
```

That `DoYouEven%sCTF` line is the whole game, or at least the opening move of it. `strings` gets you the shape of the puzzle; it doesn't get you the solution, which is exactly the point this room is making with its own tagline.

```
Static enumeration chain
┌───────────────────────────────────────────┐
│  file ──► confirms 64-bit ELF                   │
│         │                                     │
│  Execute ──► prompts for a password               │
│         │                                     │
│  strings ──► "DoYouEven%sCTF" format string         │
│         │           ──► not enough on its own       │
└───────────────────────────────────────────┘
```

---

## Static Analysis

### Decompiling With Ghidra

Loading the binary into Ghidra and letting it decompile `main()` finally shows the actual logic behind that prompt:

```c
undefined8 main(void)
{
    int iVar1;
    char local_28[32];

    fwrite("Password: ", 1, 10, stdout);
    __isoc99_scanf("DoYouEven%sCTF", local_28);

    iVar1 = strcmp(local_28, "__dso_handle");
    if ((-1 < iVar1) && (iVar1 = strcmp(local_28, "__dso_handle"), iVar1 < 1)) {
        printf("Try again!");
        return 0;
    }

    iVar1 = strcmp(local_28, "_init");
    if (iVar1 == 0) {
        printf("Correct!");
    } else {
        printf("Try again!");
    }
    return 0;
}
```

Four things matter here, and none of them are the intimidating-looking symbol names:

1. `local_28` is a 32-byte buffer, and it's what actually gets checked, not the raw input.
2. The `scanf` call uses `"DoYouEven%sCTF"` as its format string, not `"%s"` alone, which means the literal text `DoYouEven` and `CTF` in the input are consumed by the format string itself and never make it into `local_28`. Only whatever's sandwiched between them lands in the buffer.
3. The first `strcmp` block is a decoy dressed up as a real check: it compares `local_28` against `"__dso_handle"` twice in a row using a slightly odd `(-1 < iVar1) && (iVar1 < 1)` condition, which is just a convoluted way of asking "does this equal zero," i.e., "did the strings match." It's a rejection check, not the actual win condition.
4. The real win condition is the second `strcmp`, comparing `local_28` against the literal string `"_init"`. Match that, and the binary prints `Correct!`.

```
Decompiled logic chain
┌───────────────────────────────────────────┐
│  scanf("DoYouEven%sCTF", local_28)              │
│         │                                     │
│  "DoYouEven" and "CTF" consumed by format string  │
│         │                                     │
│  Only the middle portion lands in local_28         │
│         │                                     │
│  Reject if local_28 == "__dso_handle"                │
│  Accept if local_28 == "_init"                         │
└───────────────────────────────────────────┘
```

### Working Out What the Input Actually Needs to Be

Since `scanf` is parsing `"DoYouEven%sCTF"` as a literal template, not a free-form prompt, the actual text typed at the terminal has to contain `DoYouEven` and `CTF` as literal substrings surrounding whatever gets captured into `local_28`. So if the goal is for `local_28` to end up holding exactly `_init`, the full string typed into the prompt needs to be:

```
DoYouEven_initCTF
```

Except here's the twist: because `%s` in `scanf` reads everything up to the next whitespace, and the literal `CTF` in the format string is matched against literal characters in the input rather than treated as a stopping point on its own, the practical, tested answer that satisfies the room turns out to be simpler than the full literal reconstruction: just **`DoYouEven_init`**, letting the format string's trailing literal soak up whatever's left. Testing it directly against the binary confirms it:

```bash
echo "DoYouEven_init" | ./Compiled
```

```
Password: Correct!
```

```
Answer 1 — Password: DoYouEven_init
```

---

## Investigation Summary

```
[file] ──► 64-bit ELF binary
         │
[Execute] ──► prompts "Password:"
         │
[strings] ──► finds "DoYouEven%sCTF" format string
         │           (helpful, but not the full answer)
         │
[Ghidra decompile] ──► main() logic revealed
         │
[scanf format parsing] ──► literal DoYouEven / CTF consumed
         │
[strcmp checks] ──► reject "__dso_handle", accept "_init"
         │
[Reconstructed input] ──► DoYouEven_init ──► Correct!
```

---

## Flags

| # | Question | Answer |
|---|----------|--------|
| 1 | What is the password? | `DoYouEven_init` |

**Would I recommend this room?** Yes, it's a great five-minute introduction to reading decompiled C with Ghidra. It doesn't need assembly-level reversing skill, just the patience to actually trace through a `scanf` format string instead of assuming `%s` behaves the way it would in isolation. Good stepping stone before tackling a room with real anti-debugging or actual obfuscation involved.

---

## Blue Team Takeaways

**`strings` Is Reconnaissance, Not a Solution**
The format string alone was visible from the very first pass, but it only described the shape of the check, not the logic behind it. Anyone auditing an unfamiliar binary, defensively or otherwise, should treat `strings` output as a starting point for deeper static or dynamic analysis, never as the finish line.

**Format Strings Deserve Careful Reading, Not Assumptions**
`"DoYouEven%sCTF"` looks, at a glance, like a simple "read a string" prompt, but the literal text embedded in it fundamentally changes what has to be typed for the program to behave a certain way. Format string handling is a classic source of both security bugs (format string vulnerabilities in real-world C code) and simple logic misunderstandings; both deserve the same close read this challenge demanded.

**Hardcoded Comparison Strings Are Trivial to Recover Statically**
Both `"__dso_handle"` and `"_init"` sat in the binary in plain text, fully recoverable without ever running the program under a debugger. Any secret, password, key, or magic value compiled directly into a binary should be assumed fully exposed to anyone willing to decompile it; real secret validation belongs server-side or behind a properly protected credential store, never as a literal string sitting in a client-side binary.

**Decoy Logic Doesn't Slow Down Static Analysis**
The `__dso_handle` rejection branch, dressed up with a slightly convoluted double-comparison, was clearly meant to look like it mattered more than it did. Obfuscating control flow with decoy branches raises the reading effort slightly but doesn't meaningfully protect anything once a decompiler lays the whole function out in readable form.

---

*Turns out the password to this whole challenge was hiding in the compiler's own leftovers — proof that sometimes the most secure-looking symbol in your binary is just an internal linker artifact wearing a trench coat.*
