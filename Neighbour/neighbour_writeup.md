# [Neighbour](https://tryhackme.com/room/neighbour)

| Field      | Details                          |
|------------|----------------------------------|
| Room       | [Neighbour](https://tryhackme.com/room/neighbour) |
| Platform   | TryHackMe                        |
| Difficulty | Easy                              |
| Category   | Web Exploitation, Access Control |
| Tags       | IDOR, Source Code Disclosure, Broken Access Control |

---

## Overview

Authentication Anywhere promises a totally secure login process. It delivers a login process, and roughly zero percent of the security. The premise is that guests can log in from anywhere, admins keep secrets in their profile, and nothing stops a guest from simply asking to see them.

From a blue team perspective this room is a clean demonstration of Insecure Direct Object Reference, one of the most frequently exploited and most easily prevented classes of vulnerability in production web applications. No exploit chain, no payload crafting, just an application that trusts whatever value shows up in a parameter and calls it identity.

---

## Enumeration

### Port Scan

```bash
nmap -Pn -A <TARGET_IP>
```

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.5
80/tcp open  http    Apache httpd 2.4.53
```

The usual pair. SSH is a dead end without credentials, so the web app on port 80 is where this room lives and dies.

```
Attack surface
┌──────────────────────────────────────┐
│  Internet                            │
│         ┌──────────┐                 │
│         │  Port 80 │◄── Start here   │
│         │  Apache  │                 │
│         └──────────┘                 │
│         ┌──────────┐                 │
│         │  Port 22 │◄── No creds yet │
│         │  SSH     │                 │
│         └──────────┘                 │
└──────────────────────────────────────┘
```

### Web Reconnaissance

`http://<TARGET_IP>/` presents a bare bones login form for "Authentication Anywhere." Below the form is a hint that a guest account exists and that the page source is worth checking, which is about as loud as a hint can get without just handing over the credentials directly.

**View Page Source (Ctrl+U):**

```html
<!-- use guest:guest credentials until registration is fixed. "admin" user account is off limits!!!!! -->
```

The comment does two things at once. It hands over working credentials, and it tells us exactly which account is worth impersonating once we are inside. Rick left his password in robots.txt. Whoever wrote this comment left the entire attack plan in an HTML comment.

```
Information disclosure chain
┌───────────────────────────────────────────────┐
│  HTML comment ──► credentials: guest / guest   │
│  HTML comment ──► admin account named directly │
└───────────────────────────────────────────────┘
```

---

## Initial Access

### Logging in as Guest

```
Username: guest
Password: guest
```

The login succeeds and redirects to a profile style page. The URL is where things get interesting:

```
http://<TARGET_IP>/profile.php?user=guest
```

A `user` parameter sitting in plain text in the URL, controlling which profile gets rendered, is the textbook shape of an IDOR. The application is using client supplied input as a direct reference to a backend object (in this case, another user's profile) without checking whether the logged in session actually has permission to view it.

```
IDOR surface
┌────────────────────────────────────────────┐
│  Session proves: "I am guest"               │
│  URL parameter says: "show me user=guest"   │
│                                              │
│  Nothing stops user= from becoming anything  │
│  else, because the server never checks      │
│  whether session and parameter agree.        │
└────────────────────────────────────────────┘
```

---

## Privilege Escalation (via IDOR)

### Swapping the Parameter

The comment already named the target. Editing the URL directly:

```
http://<TARGET_IP>/profile.php?user=admin
```

The server renders the admin's profile page in full, including whatever the admin considers a secret, because it never verified that the guest session was entitled to request the admin's record. There is no privilege check between "who is logged in" and "whose data gets returned."

```
Exploitation chain
┌──────────────────────────────────────────────┐
│  Login as guest:guest                         │
│         │                                     │
│         ▼                                     │
│  profile.php?user=guest  ──► own profile       │
│         │                                     │
│         ▼ (edit parameter, no auth check)     │
│  profile.php?user=admin  ──► admin's profile   │
│                           ──► flag disclosed   │
└──────────────────────────────────────────────┘
```

### Capturing the Flag

The admin profile page displays the room flag directly once the parameter swap succeeds.

---

## Attack Path Summary

```
[Recon]
  │
  └─ view-source on login page ──► guest:guest creds + admin named as target
         │
[Initial Access]
  │
  └─ login as guest ──► profile.php?user=guest
         │
[IDOR Exploitation]
  │
  └─ user=guest ──► user=admin
                 ──► no server-side ownership check
                 ──► admin profile rendered
                 ──► flag captured
```

---

## Flags

| # | Question                              | Answer                                   |
|---|-----------------------------------------|-------------------------------------------|
| 1 | What is the flag found on the admin's profile? | *(value shown on your deployed instance)* |

---

## Blue Team Takeaways

**Secrets Left in HTML Comments**
Credentials, account names, and internal notes left in page source are functionally public. Anyone with a browser and Ctrl+U has them. Code review before deployment and automated secret scanning in CI/CD pipelines exist precisely to catch this before it reaches production, and no comment intended as a developer note should ever ship to a live environment.

**Insecure Direct Object Reference**
The core failure here is authorization happening nowhere. Authentication confirmed guest was logged in, but nothing then confirmed guest was allowed to request admin's data. The fix is server-side ownership checks on every object reference. Every parameter that identifies a resource (`user=`, `id=`, `account=`) should be validated against the authenticated session before any data is returned, never trusted at face value from the client.

**Why IDOR Persists in Production**
This class of bug survives because it produces no errors, no crashes, and no obviously malicious looking traffic. The request looks completely legitimate. Detection relies on access control being enforced at the data layer rather than assumed from a valid login, and on monitoring for authenticated users requesting resource identifiers that do not belong to them.

---

*Authentication Anywhere. Authorization nowhere.*
