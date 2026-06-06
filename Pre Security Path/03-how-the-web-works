# Pre-Security: Module 3 - How The Web Works

**Learning Path:** Pre-Security  
**Module:** 03 - How The Web Works  
**Rooms Completed:** DNS in Detail · HTTP in Detail · How Websites Work · Putting It All Together  
**Date Completed:** April 2026  
**Difficulty:** Beginner (Theory + Interactive Labs)

---

## Overview

Module 3 shifts focus from network infrastructure to the web specifically. DNS, HTTP, HTML, JavaScript and how a browser request actually travels from your keyboard to a server and back. This is directly relevant to web application security, which is one of the biggest areas of both offensive and defensive work in the industry.

Coming from a background in SIEM monitoring and incident response, a lot of web-layer attacks show up in logs as HTTP anomalies: unexpected methods, unusual status codes, requests to paths that should not exist. This module gave me a proper framework for understanding what normal looks like, which makes spotting abnormal much easier.

---

## Room 1 - DNS in Detail

**Room Link:** https://tryhackme.com/room/dnsindetail  
**Format:** Reading + nslookup Lab

### What It Covers

DNS stands for Domain Name System. It translates human-readable domain names like `tryhackme.com` into IP addresses that computers use to route traffic. Without DNS you would need to memorise IP addresses for every website you visit.

**Domain Hierarchy:**

- **TLD (Top Level Domain):** The rightmost part of a domain. `.com`, `.org`, `.ie`. TLDs are managed by ICANN.
- **Second Level Domain:** The name directly to the left of the TLD. In `tryhackme.com`, this is `tryhackme`. Limited to 63 characters, only `a-z`, `0-9` and hyphens.
- **Subdomain:** Sits to the left of the second level domain. In `admin.tryhackme.com`, `admin` is the subdomain. Multiple subdomains can be chained but the full domain name cannot exceed 253 characters.

**DNS Record Types:**

| Record Type | Purpose |
|-------------|---------|
| A | Maps a domain to an IPv4 address |
| AAAA | Maps a domain to an IPv6 address |
| CNAME | Maps a domain to another domain name (alias) |
| MX | Specifies mail servers for a domain |
| TXT | Stores arbitrary text, used for verification and spam prevention |

**How a DNS Request Works:**

1. Browser checks local cache first
2. If not cached, asks the Recursive DNS Server (usually your ISP or a public resolver like `8.8.8.8`)
3. Recursive server checks its own cache
4. If not cached, asks a Root DNS Server which directs to the correct TLD server
5. TLD server directs to the Authoritative Name Server for that domain
6. Authoritative server returns the actual record
7. Recursive server caches the result and returns it to your browser

TTL (Time To Live) controls how long a DNS record is cached before it expires and needs to be looked up again.

### The Lab - nslookup

The practical uses `nslookup` against a simulated DNS server to look up different record types. Commands used:

```bash
nslookup --type=CNAME shop.website.thm
nslookup --type=TXT website.thm
nslookup --type=MX website.thm
nslookup --type=A www.website.thm
```

DNS record lookups are something I have used in real investigations. When triaging phishing alerts, checking MX and TXT records (specifically SPF, DKIM and DMARC) is a standard part of validating whether a sender domain is legitimate. This lab puts those commands in proper context.

### Task Answers

| Task | Question | Answer |
|------|----------|--------|
| Task 1 | What does DNS stand for? | `Domain Name System` |
| Task 2 | What is the maximum length of a subdomain? | `63` |
| Task 2 | Which of the following characters cannot be used in a subdomain? (3 b _ -) | `_` |
| Task 2 | What is the maximum length of a domain name? | `253` |
| Task 2 | What type of TLD is .co.uk? | `ccTLD` |
| Task 3 | What type of record is used to define the IP address of a domain? | `A` |
| Task 3 | What type of record is used to define where email should be sent? | `MX` |
| Task 3 | What type of record handles email authentication? | `TXT` |
| Task 4 | What field specifies how long a DNS record should be cached? | `TTL` |
| Task 4 | What type of DNS server is responsible for the final answer? | `Authoritative` |
| Task 5 | What is the CNAME of shop.website.thm? | `shops.myshopify.com` |
| Task 5 | What is the value of the TXT record of website.thm? | `THM{7012BBA60997F35A9516C2E16D2944FF}` |
| Task 5 | What is the numerical priority value for the MX record? | `30` |
| Task 5 | What is the IP address for www.website.thm? | `10.10.10.10` |

---

## Room 2 - HTTP in Detail

**Room Link:** https://tryhackme.com/room/httpindetail  
**Format:** Reading + Request/Response Lab

### What It Covers

HTTP (HyperText Transfer Protocol) is the protocol browsers use to communicate with web servers. HTTPS is the encrypted version using TLS. The difference is critical from a security perspective: HTTP traffic is plaintext and can be intercepted and read by anyone on the network. HTTPS encrypts the data so interception only reveals ciphertext.

**HTTP Methods:**

| Method | Purpose |
|--------|---------|
| GET | Retrieve data from a server |
| POST | Send data to a server (login forms, file uploads) |
| PUT | Update existing data on a server |
| DELETE | Remove data from a server |

**HTTP Status Codes:**

| Range | Meaning |
|-------|---------|
| 100-199 | Informational |
| 200-299 | Success |
| 300-399 | Redirection |
| 400-499 | Client errors |
| 500-599 | Server errors |

Common ones worth knowing: `200 OK`, `201 Created`, `301 Moved Permanently`, `302 Found (redirect)`, `400 Bad Request`, `401 Unauthorised`, `403 Forbidden`, `404 Not Found`, `500 Internal Server Error`.

**Headers** carry additional information in requests and responses. Important ones:

- `Host` - which website is being requested (required for virtual hosting)
- `User-Agent` - identifies the browser and OS
- `Content-Type` - what format the body data is in
- `Set-Cookie` / `Cookie` - how sessions are maintained between requests
- `Location` - used in redirect responses

**Cookies** are small pieces of data the server sends to the browser and the browser sends back with every subsequent request. They are how websites maintain session state. HTTP is stateless by design so without cookies every request would be treated as brand new with no memory of a previous login.

From a security perspective cookies are a significant attack surface. Weak cookie implementations lead to session hijacking, CSRF vulnerabilities and auth bypass. The `Secure` flag ensures cookies are only sent over HTTPS. The `HttpOnly` flag prevents JavaScript from accessing them, which mitigates XSS-based cookie theft.

### The Lab - HTTP Request Simulation

An interactive request builder where you change the method, path and parameters and see the raw HTTP request and response update live. Each configuration reveals a flag.

### Task Answers

| Task | Question | Answer |
|------|----------|--------|
| Task 1 | What does HTTP stand for? | `HyperText Transfer Protocol` |
| Task 1 | What does the S in HTTPS stand for? | `Secure` |
| Task 1 | What is the flag from the HTTPS lab? | `THM{HTTPS_CERT_IS_GREAT}` |
| Task 2 | What HTTP protocol is being used in the above example? | `HTTP/1.1` |
| Task 2 | What response header tells the browser how much data to expect? | `Content-Length` |
| Task 3 | What method is used to retrieve a web page? | `GET` |
| Task 3 | What method is used to send data to a web server? | `POST` |
| Task 3 | What method is used to update data on a web server? | `PUT` |
| Task 3 | What method is used to delete data from a web server? | `DELETE` |
| Task 4 | What response code means the request was successful? | `200` |
| Task 4 | What response code means a new resource has been created? | `201` |
| Task 4 | What response code means you need to authenticate? | `401` |
| Task 4 | What response code means you do not have permission? | `403` |
| Task 4 | What response code means the page does not exist? | `404` |
| Task 4 | What response code means the server has an error? | `500` |
| Task 5 | What header tells the browser which website is being requested? | `Host` |
| Task 5 | What header identifies the client's browser? | `User-Agent` |
| Task 5 | What header tells the server what type of data is in the request body? | `Content-Type` |
| Task 6 | Which header is used to save cookies to the browser? | `Set-Cookie` |
| Task 7 | Make a GET request to /room | `THM{YOU'RE_IN_THE_ROOM}` |
| Task 7 | Make a GET request to /blog with id=1 | `THM{YOU_FOUND_THE_BLOG}` |
| Task 7 | Make a DELETE request to /user/1 | `THM{USER_IS_DELETED}` |
| Task 7 | Make a PUT request to /user/2 with username=admin | `THM{USER_HAS_UPDATED}` |
| Task 7 | POST username=thm and password=letmein to /login | `THM{HTTP_REQUEST_MASTER}` |

---

## Room 3 - How Websites Work

**Room Link:** https://tryhackme.com/room/howwebsiteswork  
**Format:** Reading + Interactive HTML/JS/Injection Labs

### What It Covers

Websites have two sides: the front end (what your browser renders) and the back end (the server processing requests and returning data). This room focuses on the front end: HTML, JavaScript and two common vulnerabilities that arise when front end code is handled carelessly.

**HTML** is the structure of a web page. Tags define elements: headings, paragraphs, images, links, buttons. The browser reads HTML and renders it visually. If you right-click any webpage and select View Source you see the raw HTML the server sent.

**JavaScript** makes pages interactive. It runs in the browser and can manipulate the page content dynamically without reloading. The `document.getElementById()` function is used to target specific HTML elements and change their content or behaviour.

**Sensitive Data Exposure** happens when developers accidentally leave sensitive information in the HTML source code, in JavaScript files or in comments. Passwords, API keys and internal paths are commonly found this way. Always check the page source during a web application pentest. The room demonstrates this with a password hidden in the HTML source: `testpasswd`.

**HTML Injection** is a client-side vulnerability where unsanitised user input is rendered as HTML on the page. If a site takes user input and outputs it directly to the page without stripping HTML tags, an attacker can inject their own HTML. A link like `<a href="http://hacker.com">Click me</a>` injected into a name field would appear as a real clickable link for other users. The general rule: never trust user input, always sanitise before rendering.

### Screenshots

**JavaScript Lab - Flag JSISFUN**

The JavaScript task asks you to add code to change a page element's content to "Hack the Planet". The lab confirms the flag via a popup.

![JavaScript lab showing flag JSISFUN](../screenshots/Screenshot_2026-04-28_225159.png)

**HTML Injection Lab - Flag HTML_INJ3CT10N**

The HTML injection task asks you to inject a malicious anchor tag linking to `http://hacker.com`. Successfully injecting it triggers the flag.

![HTML Injection lab showing flag HTML_INJ3CT10N](../screenshots/Screenshot_2026-04-28_231120.png)

### Task Answers

| Task | Question | Answer |
|------|----------|--------|
| Task 1 | What term describes the component rendered by your browser? | `Front End` |
| Task 2 | One of the images on the page has broken. What is the image file extension? | `.jpg` |
| Task 3 | What is the flag from the JavaScript lab? | `JSISFUN` |
| Task 4 | What is the password hidden in the source code? | `testpasswd` |
| Task 5 | What is the flag from the HTML injection lab? | `HTML_INJ3CT10N` |

---

## Room 4 - Putting It All Together

**Room Link:** https://tryhackme.com/room/puttingitalltogether  
**Format:** Reading + Drag and Drop Quiz

### What It Covers

This room ties together everything from Module 3 by walking through the complete sequence of events that happen when you type a URL into a browser and press enter. End to end.

**The full request flow:**

1. Browser checks local cache for the DNS record
2. If not cached, recursive DNS resolver is queried
3. Resolver queries root servers, TLD servers and authoritative servers in sequence
4. IP address is returned and cached with TTL
5. Browser initiates a TCP connection to the server on port 80 (HTTP) or 443 (HTTPS)
6. For HTTPS, a TLS handshake occurs to establish an encrypted session
7. Browser sends an HTTP GET request
8. Request may pass through a load balancer if the site has multiple servers
9. Request may pass through a Web Application Firewall (WAF) which inspects for malicious patterns
10. Web server receives the request and passes it to the application
11. Application may query a database for dynamic content
12. Response is built and sent back through the same path
13. Browser renders the HTML, CSS and JavaScript

**Other components mentioned:**

- **CDN (Content Delivery Network):** Caches static content geographically close to users to reduce latency
- **Load Balancer:** Distributes incoming traffic across multiple servers to prevent any one server being overwhelmed
- **WAF (Web Application Firewall):** Sits in front of web servers and filters out known malicious request patterns

Understanding this full flow is essential for both web pentesting and defensive monitoring. At each stage there is something that can be logged, monitored or attacked.

### Screenshot

**Putting It All Together Quiz - Flag THM{YOU_GOT_THE_ORDER}**

The quiz asks you to drag tiles into the correct order showing how a browser request flows end to end. Getting it right reveals the flag.

![Putting it all together quiz with flag THM{YOU_GOT_THE_ORDER}](../screenshots/Screenshot_2026-04-28_232156.png)

### Task Answers

| Task | Question | Answer |
|------|----------|--------|
| Task 2 | What is the name of the component that checks website requests against a list of rules? | `Web Application Firewall` |
| Task 2 | What is the name of the component used to distribute load across multiple servers? | `Load Balancer` |
| Task 3 | What does a web server use to host multiple websites on one IP? | `Virtual Hosts` |
| Task 4 | What is the flag from the quiz? | `THM{YOU_GOT_THE_ORDER}` |

---

## What I Learned / Reinforced

**DNS is more than just name resolution.** TXT records carry SPF, DKIM and DMARC data used for email authentication. MX records point to mail servers. CNAME records create aliases. All of these are checked during phishing investigations and domain reconnaissance. Understanding DNS record types is practical knowledge, not just exam content.

**HTTP methods matter for security.** A web app that accepts DELETE or PUT requests from unauthenticated users has a serious problem. Understanding what each method is supposed to do makes it obvious when something is being misused. SIEM rules that alert on unexpected HTTP methods to sensitive paths are a real detection pattern.

**Sensitive data exposure is embarrassingly common.** The room uses a simple example but in real web app assessments finding credentials or API keys in HTML comments or JavaScript files is not rare. Automated scanners like Burp Suite look for exactly this. Developers forget that client-side code is fully readable by anyone.

**HTML injection is XSS with the volume turned down.** Injected HTML can redirect users, create convincing phishing links within a trusted site and manipulate what other users see. Full XSS goes further by executing JavaScript but the root cause is the same: unvalidated user input reaching the page. Input sanitisation is non-negotiable.

**The full request flow is the mental model for web security.** DNS poisoning attacks at step 2. TLS stripping attacks at step 6. SQL injection attacks at step 11. Understanding the flow lets you map attack techniques to the exact stage they target.

---

## Resources

- [DNS in Detail - TryHackMe](https://tryhackme.com/room/dnsindetail)
- [HTTP in Detail - TryHackMe](https://tryhackme.com/room/httpindetail)
- [How Websites Work - TryHackMe](https://tryhackme.com/room/howwebsiteswork)
- [Putting It All Together - TryHackMe](https://tryhackme.com/room/puttingitalltogether)
- [OWASP Top 10](https://owasp.org/www-project-top-ten/)
- [Burp Suite Community Edition](https://portswigger.net/burp/communitydownload)

---

*Written by fiza.sk293 · [GitHub](https://github.com/FizaShaikh293/thm-writeups)*
