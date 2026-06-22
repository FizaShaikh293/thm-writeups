---
layout: pickle-rick
title: Pickle Rick Writeup
---

# 🥒 Pickle Rick

<div class="info-card">

[INFO CARD]
Platform   : TryHackMe
Difficulty : Easy
Category   : Web Exploitation | Linux PrivEsc
Tags       : Command Injection | Web Shell | Sudo Misconfiguration

</div>

---

## Overview

Rick has turned himself into a pickle. Again. To undo the damage he needs three secret ingredients, but he has also managed to forget the password to his own computer. Morty is unavailable, so that leaves us. The goal is to exploit a web server, gain command execution, and recover all three ingredients before Rick spends the rest of his life as a condiment.

From a blue team perspective this room covers the complete kill chain of a web application compromise: credential exposure through source code and robots.txt, unauthenticated command injection via a web shell panel, and a catastrophically misconfigured sudo policy that hands over root with zero resistance.

---

## Enumeration

### Port Scan

```bash
nmap -sC -sV <TARGET_IP>
