# [MD2PDF](https://tryhackme.com/room/md2pdf)

| Field      | Details                          |
|------------|----------------------------------|
| Room       | [MD2PDF](https://tryhackme.com/room/md2pdf) |
| Platform   | TryHackMe                        |
| Difficulty | Easy / Medium                       |
| Category   | Web Exploitation                    |
| Tags       | SSRF, HTML Injection, Internal Service Access, Directory Enumeration |

---

## Overview

*"TopTierConversions LTD is proud to present its latest product launch."* Ladies and gentlemen, presenting: a Markdown-to-PDF converter with the access control philosophy of a nightclub bouncer who checks IDs at the front door but leaves the fire exit propped open with a brick. TopTierConversions really went all in on the launch energy, corporate tagline and everything, for a product whose core feature turns out to be "will render literally any HTML you hand it, no questions asked."

The pitch is simple: paste some Markdown, hit convert, get a PDF. The problem is that "some Markdown" quietly includes raw HTML, and raw HTML includes `<iframe>`, and an `<iframe>` pointed at the right internal address turns a document converter into a backstage pass. Somewhere in the building, port 5000 is running an admin panel that trusts anyone asking from `localhost`, blissfully unaware that the PDF renderer counts as "anyone."

```
Attack surface
┌─────────────────────────────────────┐
│  22/tcp   – SSH                        │
│  80/tcp   – HTTP (MD2PDF converter,    │
│             public-facing)              │
│  5000/tcp – Internal admin service      │
│             (trusts localhost only)     │
└─────────────────────────────────────┘
```

---

## Enumeration

### Initial Scan

```bash
nmap -p- -sV -T4 <TARGET_IP>
```

```
PORT     STATE SERVICE
22/tcp   open  ssh
80/tcp   open  http
5000/tcp open  http
```

Two web services for the price of one launch event. Port 80 is the public-facing product everyone's invited to; port 5000 is the after-party nobody outside the building was supposed to hear about.

### The Product Itself

Visiting port 80 turns up exactly what the tagline promised: a clean, minimal interface with a text box for Markdown and a **Convert to PDF** button. Feeding it basic formatting, bold text, headers, the usual, works exactly as advertised. No visible dev tools, no exposed JS files, no hidden fields helpfully labeled "vulnerability here." TopTierConversions built a genuinely functional product, which is almost more embarrassing given what's about to happen to it.

### Finding the After-Party

A quick directory brute-force against port 80 turns up something the front page never linked to:

```bash
gobuster dir -u http://<TARGET_IP> -w /usr/share/wordlists/dirb/common.txt
```

```
/admin
```

Visiting `/admin` directly gets politely, firmly declined, the kind of 403 that reads like "you're not on the list." Which is, notably, not the same thing as "this page doesn't exist." Port 5000 answers the same story: an internal-looking service that clearly expects requests to come from somewhere inside the building, not from a browser out on the public internet.

```
Enumeration chain
┌───────────────────────────────────────────┐
│  Port 80 ──► public Markdown-to-PDF product   │
│  Port 5000 ──► internal admin service           │
│         │                                     │
│  Gobuster on port 80 ──► /admin (403, exists)   │
│         │                                     │
│  Direct access denied ──► needs to look local   │
└───────────────────────────────────────────┘
```

---

## Exploiting the SSRF

### The Markdown That Isn't Just Markdown

Most Markdown-to-PDF converters, this one included, quietly support raw HTML passthrough as a "feature," because pure Markdown can't do everything, and somebody, somewhere, decided the fix for that was "just let HTML through too." That's the whole vulnerability, dressed up as a convenience.

Since the rendering happens server-side, and the server is presumably standing right there in the same building as the admin panel on port 5000, an `<iframe>` embedded in the submitted Markdown gets fetched from the server's own point of view, not the attacker's:

```html
<iframe src="http://localhost:5000/admin" width="800" height="600"></iframe>
```

### Submitting the Payload

Pasting that into the converter and hitting **Convert to PDF** does exactly what a product launch should never do on day one: it works flawlessly, just for the wrong audience. The rendering engine (commonly `wkhtmltopdf` in rooms built like this one) dutifully fetches `localhost:5000/admin` as part of building the PDF, and because that request originates from the server itself, the internal service's "localhost only" access control waves it straight through.

```
SSRF exploitation chain
┌───────────────────────────────────────────┐
│  Markdown input ──► raw HTML passthrough        │
│         │                                     │
│  <iframe src="http://localhost:5000/admin">     │
│         │                                     │
│  Server-side renderer fetches the iframe         │
│         │                                     │
│  Request appears to come from localhost           │
│         │                                     │
│  Admin panel's IP check ──► trusts it, no questions │
└───────────────────────────────────────────┘
```

### Opening the PDF

Downloading the resulting PDF and opening it up reveals exactly what the browser was denied minutes earlier: the full `/admin` dashboard, rendered inline inside the document, flag and all. The product converted Markdown to PDF exactly as promised. It just also converted "internal-only" into "publicly downloadable," which wasn't on the feature list.

```
Answer 1 — Flag: flag{1f4a2b6ffeaf4707c43885d704eaee4b}
```

---

## Investigation Summary

```
[nmap scan]
  │
  └─ 22/ssh, 80/http (public), 5000/http (internal)
         │
[Port 80] ──► Markdown-to-PDF converter
         │
[Gobuster] ──► /admin (403 from outside)
         │
[Port 5000] ──► admin panel, trusts localhost only
         │
[Markdown → raw HTML passthrough]
         │
[<iframe src="http://localhost:5000/admin">]
         │
[Server-side render fetches iframe as localhost] ──► bypasses IP check
         │
[PDF downloaded] ──► admin panel rendered inline ──► flag
```

---

## Flags

| # | Question | Answer |
|---|----------|--------|
| 1 | What is the flag? | `flag{1f4a2b6ffeaf4707c43885d704eaee4b}` |

**Would I recommend this room?** Yes, it's a great practical SSRF room precisely because there's no port scanning trickery or obscure CVE involved, just a document converter doing exactly what document converters do (fetch remote resources to render them) in a context where that behavior turns into a backstage pass. Solid room for understanding why "internal-only" access controls based on source IP fall apart the moment something inside the network can be tricked into making requests on your behalf.

---

## Blue Team Takeaways

**Server-Side Rendering Is a Request on Your Server's Behalf**
The PDF renderer fetched `localhost:5000/admin` because it was told to, and it had every bit of network access the server itself had. Any feature that renders or fetches remote content server-side, PDF generation, link previews, image proxies, webhooks, needs to be treated as a potential outbound request an attacker fully controls, not a trusted internal action.

**"Localhost Only" Is Not the Same as "Authenticated"**
The admin panel's entire security model was "did this request come from 127.0.0.1," which is a location check, not an identity check. Anything that can be tricked into making a local request, a renderer, a webhook handler, a proxy, inherits that trust instantly. Internal services should authenticate callers properly, not just check where the packet appears to originate from.

**Markdown-to-HTML Passthrough Needs a Sanitizer, Not a Shrug**
Letting raw HTML ride along inside "just Markdown" input handed over the exact primitive (`<iframe>`) needed to pull this off. If a Markdown renderer supports HTML passthrough as a feature, that HTML needs to be sanitized and stripped of anything capable of triggering outbound requests, iframes, images with remote src, script tags, before it ever reaches the rendering engine.

**Network Segmentation Should Assume Internal Services Get Reached Anyway**
Port 5000 being unreachable from outside didn't actually protect it, because the public-facing service on port 80 was perfectly capable of reaching it on the admin panel's behalf. Sensitive internal services need real authentication and authorization even when segmentation makes them "unreachable" on paper; SSRF exists specifically to prove that paper wrong.

---

*TopTierConversions LTD promised a product launch and delivered one, just not the one they were expecting: a live demo of how "internal-only" and "convert this file for me" don't mix.*
