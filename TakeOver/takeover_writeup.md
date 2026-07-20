# [TakeOver](https://tryhackme.com/room/takeover)

| Field      | Details                          |
|------------|----------------------------------|
| Room       | [TakeOver](https://tryhackme.com/room/takeover) |
| Platform   | TryHackMe                        |
| Difficulty | Easy                             |
| Category   | Web Recon, DNS Misconfiguration  |
| Tags       | Subdomain Enumeration, Virtual Host Fuzzing, SSL Certificate Analysis, Subdomain Takeover |

---

## Overview

futurevera.thm is a space research startup that is "rebuilding their support" and apparently rebuilding it in public. Some blackhat crew slid into their DMs claiming they can take the site over and want a ransom for staying quiet about it. Our job is to figure out exactly what they meant before they cash in on it.

There is no exploit here in the traditional sense. No RCE, no priv esc, no shells. Just DNS doing what DNS does when nobody double checks it: pointing a subdomain at infrastructure that no longer exists. From a blue team perspective this room is a clean case study in attack surface that lives outside the main application entirely. Virtual hosts nobody documented, a TLS certificate that leaks more than it should, and a CNAME record aimed at a cloud resource that was deleted and never cleaned up. None of that shows up in a normal vulnerability scan of the main site.

---

## Enumeration

### Port Scan

```
nmap -sC -sV futurevera.thm
```

```
PORT    STATE SERVICE
22/tcp  open  ssh
80/tcp  open  http
443/tcp open  https
```

Standard web box. SSH for later maybe, 80 and 443 are where the actual room lives.

```
Attack surface
┌───────────────────────────────────────────┐
│  futurevera.thm                            │
│         ┌──────────┐                       │
│         │  Port 22 │◄── noted, not needed  │
│         └──────────┘                       │
│         ┌──────────┐                       │
│         │  Port 80 │◄── redirects/vhosts   │
│         └──────────┘                       │
│         ┌──────────┐                       │
│         │ Port 443 │◄── TLS cert = clue     │
│         └──────────┘                       │
└───────────────────────────────────────────┘
```

### Hosts File

TryHackMe hands out the target IP for the domain, not a resolvable DNS record, so it needs to be pinned manually:

```bash
echo "<TARGET_IP> futurevera.thm" | sudo tee -a /etc/hosts
```

Visiting `https://futurevera.thm` shows a landing page for the company. The pitch mentions they are "rebuilding our support," which is a pretty direct hint that a `support` subdomain exists somewhere, intentionally or not.

### Virtual Host Fuzzing

Rather than guess one subdomain at a time, fuzz the `Host` header directly against the target IP:

```bash
ffuf -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt \
     -u https://<TARGET_IP> \
     -H "Host: FUZZ.futurevera.thm" \
     -fs 4605
```

`-fs 4605` filters out the default "unknown vhost" response size so only real matches show up.

```
blog     [Status: 200]
support  [Status: 200]
```

Two subdomains land. Both get pinned to `/etc/hosts`:

```bash
echo "<TARGET_IP> blog.futurevera.thm"    | sudo tee -a /etc/hosts
echo "<TARGET_IP> support.futurevera.thm" | sudo tee -a /etc/hosts
```

`blog.futurevera.thm` is just the blog, dead end for the room's purposes. `support.futurevera.thm` is where it gets interesting.

```
Subdomain discovery chain
┌──────────────────────────────────────────┐
│  "rebuilding our support" hint            │
│         │                                │
│  ffuf vhost fuzzing (Host: FUZZ.futurevera.thm)
│         │                                │
│         ├─► blog.futurevera.thm    (dead end)
│         └─► support.futurevera.thm (keep going)
└──────────────────────────────────────────┘
```

---

## Initial Access

### Reading the Certificate

`support.futurevera.thm` serves over HTTPS with a valid-looking TLS certificate. Certificates carry a Subject Alternative Name (SAN) field listing every hostname they are valid for, and this one lists more than just `support.futurevera.thm`.

Pulling the cert straight from the command line instead of clicking through a browser padlock:

```bash
openssl s_client -connect support.futurevera.thm:443 -servername support.futurevera.thm </dev/null 2>/dev/null | openssl x509 -noout -text | grep -A1 "Subject Alternative Name"
```

The SAN field reveals a third, unlisted hostname buried under the support subdomain, something like `secrethelpdesk<random>.support.futurevera.thm`. This was never in any wordlist because it isn't guessable, it just leaked straight out of the cert.

```bash
echo "<TARGET_IP> secrethelpdesk<random>.support.futurevera.thm" | sudo tee -a /etc/hosts
```

```
Certificate leak chain
┌──────────────────────────────────────────────┐
│  TLS cert for support.futurevera.thm          │
│         │                                     │
│  SAN field lists extra hostname                │
│         │                                     │
│  secrethelpdesk<random>.support.futurevera.thm │
│         │                                     │
│  Added to /etc/hosts ──► visit it directly     │
└──────────────────────────────────────────────┘
```

### The Dangling Subdomain

Hitting the hidden hostname on **port 80** (not 443, the redirect only fires on plain HTTP) sends the browser somewhere unexpected: an AWS S3 static website endpoint. The bucket it points to doesn't exist anymore.

```
http://secrethelpdesk<random>.support.futurevera.thm/
  ──► redirects to ──►
http://flag{...}.s3-website-us-west-3.amazonaws.com/
  ──► NoSuchBucket
```

That is the whole vulnerability sitting right there in the address bar. Someone spun up an S3 bucket, pointed a CNAME at it, then deleted the bucket and forgot the DNS record still points at it. S3 bucket names are globally unique and first-come-first-served, so anyone who notices this can register that exact bucket name and start serving whatever content they want under `support.futurevera.thm`'s good name. TLS cert, trusted domain, and all.

The flag in this room is baked directly into the bucket name that the CNAME was pointing at, so it shows up in the redirected URL itself, no further exploitation required to view it. Actually claiming the bucket (creating it in your own AWS account to serve content) is the real-world next step and is what "takeover" means, even though the room itself just wants you to spot and report it.

---

## Attack Path Summary

```
[Recon]
  │
  ├─ nmap ──► 22/80/443 open
  └─ "rebuilding our support" hint ──► guess support subdomain exists
         │
[Subdomain Enumeration]
  │
  └─ ffuf vhost fuzzing ──► blog.futurevera.thm, support.futurevera.thm
         │
[Certificate Analysis]
  │
  └─ TLS SAN field on support.futurevera.thm ──► hidden hostname leaked
         │
[Subdomain Takeover]
  │
  └─ hidden host on port 80 ──► CNAME points to deleted S3 bucket
                              ──► dangling DNS, bucket name in redirect URL
                              ──► flag
```

---

## Flags

| # | Question                                   | Answer                                  |
|---|---------------------------------------------|------------------------------------------|
| 1 | What can be taken over?                     | The dangling `support` subdomain pointing to a deleted, re-claimable S3 bucket |
| 2 | What is the flag?                           | `flag{...}` (unique per deployment, appears in the S3 redirect URL) |

---

## Blue Team Takeaways

**Hidden Virtual Hosts Are Still Attack Surface**
`blog` and `support` never showed up on the main landing page but both resolved fine once the Host header pointed at them. Anything that answers on your IP is in scope for an attacker regardless of whether it is linked anywhere. Vhost enumeration should be part of routine external attack surface reviews, not just something pentesters do.

**TLS Certificates Leak Infrastructure**
The SAN field exists to make certificates valid for multiple hostnames, but it also hands out a list of every hostname that cert covers, including ones nobody meant to advertise. Certificate Transparency logs make this worse at internet scale since every publicly issued cert gets logged permanently and is searchable by anyone. Defensive teams should assume anything in a SAN, and anything ever issued a public cert, is discoverable.

**Dangling DNS Records Enable Subdomain Takeover**
A CNAME or alias pointed at a cloud resource that gets deleted without also deleting the DNS record is a textbook subdomain takeover setup. Whoever claims that resource name next inherits a trusted subdomain, complete with a valid TLS cert if HTTPS is in play. This is exploited constantly in the wild against exactly this pattern: S3 buckets, Azure blobs, Heroku apps, GitHub Pages, abandoned CNAMEs pointing at expired third-party services. DNS hygiene during decommissioning, not just provisioning, is the fix. Automated dangling record scanners (like `dnsrecon`, `Sub404`, or cloud-native equivalents) should run continuously against production zones, not just once during onboarding.

---

*Turns out the ransom demand was just a company reading its own DNS zone file back to itself.*
