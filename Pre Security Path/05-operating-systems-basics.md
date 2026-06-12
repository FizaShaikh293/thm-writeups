---
layout: default
title: Operating Systems Basics
---

**Learning Path:** Pre-Security

**Module:** 05 - Operating Systems Basics

**Rooms Completed:** Operating Systems: Introduction · Windows Basics · Linux CLI Basics · Windows CLI Basics · Operating System Security

**Date Completed:** May - June 2026

**Difficulty:** Beginner (Theory + Interactive Labs)

---

## Overview

Every piece of software a security analyst uses runs on top of an operating system. Every log you read, every alert you triage, every shell you pop lands you inside an OS. Module 5 is the foundation beneath everything else. It covers what an OS actually does, how Windows and Linux are navigated from both GUI and command line, and then finishes with the security implications of weak accounts and poor OS hardening.

From a SOC perspective, this module is directly applicable. SIEM alerts reference process names, file paths and user accounts. Understanding how an OS organises these, and how attackers abuse them, is what separates someone who reads an alert from someone who understands it.

---

## Room 1 - Operating Systems: Introduction

**Room Link:** https://tryhackme.com/room/operatingsystemsintroduction
**Format:** Reading + Interactive Guided Lab

### What It Covers

An operating system is the software layer that sits between hardware and user applications. Without it, every application would need to communicate directly with hardware, causing conflicts and chaos. The OS acts as a central organiser handling four core responsibilities:

```
+---------------------------+
|     User Applications     |
+---------------------------+
|    Operating System       |
|  (Kernel + User Space)    |
+---------------------------+
|     Physical Hardware     |
|   CPU · RAM · Storage     |
+---------------------------+
```

**Core OS Responsibilities:**

| Responsibility | What It Does |
|---|---|
| Process Management | Allocates CPU time to running programs, handles scheduling |
| Memory Management | Controls which programs access which regions of RAM |
| File System Management | Organises data on disk into files and directories |
| Device Management | Handles communication with hardware via drivers |
| User Management | Controls user accounts, authentication and permissions |
| Security | Enforces access controls and isolation between processes |

**Kernel Space vs User Space:**

The OS is split into two privilege zones. The kernel runs in kernel space with unrestricted hardware access. It is the core of the OS. User applications run in user space with limited access, and must request kernel services via system calls. This separation is a security boundary. Malware that achieves kernel-level execution (rootkits) is significantly harder to detect and remove because it operates below the user space security tooling.

**OS Types:**

| Category | Examples | Where Used |
|---|---|---|
| Desktop/Workstation | Windows 11, macOS, Ubuntu Desktop | End-user machines |
| Server | Windows Server, Ubuntu Server, Red Hat | Data centres, web hosting |
| Mobile | Android, iOS | Phones, tablets |
| Embedded | OpenWrt, FreeRTOS | Routers, industrial controllers |
| Cloud/VM | Amazon Linux, Rocky Linux | Cloud infrastructure |

### Task Answers

| Task | Question | Answer |
|---|---|---|
| Task 1 | Ready to start? | No answer needed |
| Task 2 | Which OS space has unrestricted access to hardware? | Kernel Space |
| Task 2 | Which OS responsibility manages user accounts, authentication and permissions? | User Management |
| Task 3 | After opening the About This Computer shortcut, what OS is running? | (Varies - read from the lab VM) |

---

## Room 2 - Windows Basics

**Room Link:** https://tryhackme.com/room/windowsbasics
**Format:** Interactive Windows VM Lab

### What It Covers

Windows is the dominant OS in enterprise environments. Most corporate endpoints run Windows. Active Directory, Group Policy and Windows Event Logs are all things a SOC analyst works with daily. This room gets you comfortable navigating Windows as a new user.

**Key Navigation Points:**

The Settings app is the modern configuration hub. Start > Settings covers everything from display to accounts to Windows Update. The older Control Panel still exists for legacy settings.

About your PC (Settings > System > About) shows:
- Device name and processor
- Installed RAM
- Windows edition and version (build number)
- Whether Windows Defender is active

**File System Navigation:**

Windows organises files under drive letters (C:, D: etc). Key locations:

```
C:\
├── Users\
│   └── [username]\
│       ├── Desktop\
│       ├── Documents\
│       ├── Downloads\
│       └── AppData\     (hidden - stores application config)
├── Windows\
│   ├── System32\        (core OS files and executables)
│   └── SysWOW64\        (32-bit compatibility layer on 64-bit systems)
└── Program Files\
    └── Program Files (x86)\
```

**Installing and Removing Software:**

Windows has two main software channels:
- Microsoft Store: sandboxed apps, auto-updates
- Traditional installers (.exe, .msi): more control, more risk

Uninstalling via Settings > Apps or Control Panel > Programs removes entries from the Add/Remove Programs list. Attackers sometimes manually delete files to stay off this list.

**Task Answers**

| Task | Question | Answer |
|---|---|---|
| Task 1 | No answer needed | - |
| Task 2 | What is the Windows edition on the lab machine? | (Read from About your PC in the lab VM) |
| Task 3 | Complete the app installation task | (Follow lab steps) |
| Task 3 | What is the flag from the TryHatMe installation? | (Generated during install - check lab) |

> Note: The Windows Basics room uses a live VM. Several answers come directly from reading system information inside the VM rather than having a fixed universal answer. Complete the lab and read the values from the machine itself.

---

## Room 3 - Linux CLI Basics

**Room Link:** https://tryhackme.com/room/linuxclibasics
**Format:** Interactive Linux VM (Story-Based - Intern Mission)

### What It Covers

Linux runs the majority of web servers, cloud infrastructure and security tooling. Almost every offensive security tool runs on Linux. Kali Linux, the go-to pentest OS, is Linux. The THM AttackBox is Linux. If you are going to do anything beyond beginner-level security work, you need to be comfortable in a Linux terminal.

This room takes you through the basics as an intern completing a mission.

**Essential Linux Commands:**

```bash
# Navigation
pwd           # Print working directory (where am I?)
ls            # List directory contents
ls -la        # Long format, including hidden files (starting with .)
cd /path      # Change to absolute path
cd ..         # Go up one directory

# File Operations
cat filename  # Print file contents
find / -name "filename"   # Search entire system for a file
find /home -name "*.txt"  # Search for all .txt files under /home

# System Info
whoami        # Current logged-in username
uname -a      # Full system/kernel information
hostname      # Machine hostname
```

**Finding the Mission Brief (Lab Walkthrough):**

The room gives you a file to find. Using the `find` command is the right approach:

```bash
find / -name "mission_brief.txt" 2>/dev/null
```

The `2>/dev/null` redirects permission errors so the output stays clean. The file is located at:

```
/home/ubuntu/Documents/.research/archive/mission_brief.txt
```

The `.research` folder is hidden (starts with a dot). `ls` without the `-a` flag would not show it. This is a common pattern in CTFs and in real incidents where attackers hide files in dot-directories.

Reading it:

```bash
cat /home/ubuntu/Documents/.research/archive/mission_brief.txt
```

**Task Answers**

| Task | Question | Answer |
|---|---|---|
| Task 1 | What does CLI stand for? | command-line interface |
| Task 2 | What is the full path of mission_brief.txt? | /home/ubuntu/Documents/.research/archive/mission_brief.txt |
| Task 2 | What is the flag inside mission_brief.txt? | MISSION-FOUND |
| Task 3 | What is the username from whoami? | ubuntu |
| Task 3 | What is the kernel version from uname -a? | (Run the command in the lab VM - varies) |

---

## Room 4 - Windows CLI Basics

**Room Link:** https://tryhackme.com/room/windowsclibasics
**Format:** Interactive Windows VM (Story-Based)

### What It Covers

The Windows Command Prompt (cmd.exe) is the command line interface built into every version of Windows. Attackers use it. Defenders use it. Incident responders use it. Being able to navigate a Windows system without a GUI, and knowing what commands an attacker might leave in command history, is a genuine SOC skill.

**Essential Windows CMD Commands:**

```cmd
REM Navigation
cd                      # Print current directory (same as pwd on Linux)
cd C:\Users             # Navigate to path
cd ..                   # Go up one directory
dir                     # List directory contents
dir /a                  # List including hidden files

REM File Search and Reading
dir /s /b task_brief.txt         # Search recursively for a file
type C:\path\to\file.txt         # Print file contents (like cat on Linux)

REM System Information
systeminfo               # Full system info (OS, RAM, hotfixes)
whoami                   # Current user
hostname                 # Machine name
ipconfig                 # Network configuration
ipconfig /all            # Detailed network config including MAC address
```

**Linux vs Windows CLI Comparison:**

```
+------------------+------------------+------------------+
|    Task          |   Linux (bash)   |  Windows (cmd)   |
+------------------+------------------+------------------+
| Where am I?      | pwd              | cd               |
| List files       | ls               | dir              |
| Read a file      | cat file         | type file        |
| Search for file  | find / -name X   | dir /s /b X      |
| Current user     | whoami           | whoami           |
| System info      | uname -a         | systeminfo       |
+------------------+------------------+------------------+
```

**Finding task_brief.txt (Lab Walkthrough):**

```cmd
dir /s /b task_brief.txt
```

The `/s` flag searches all subdirectories. The `/b` flag gives bare output (just the path, no formatting). Once you find the path, read it:

```cmd
type C:\Users\[path]\task_brief.txt
```

**Task Answers**

| Task | Question | Answer |
|---|---|---|
| Task 1 | No answer needed | - |
| Task 2 | Navigate and find the task brief | (Follow lab steps) |
| Task 2 | What is the content of task_brief.txt? | (Read in the lab VM) |
| Task 3 | What command shows detailed network info including MAC? | ipconfig /all |
| Task 3 | What command shows full OS and RAM information? | systeminfo |

> Note: Like Windows Basics, several answers require running commands inside the lab VM and reading the output directly.

---

## Room 5 - Operating System Security

**Room Link:** https://tryhackme.com/room/operatingsystemsecurity
**Format:** Reading + SSH Lab (Linux)

### What It Covers

This room ties together OS concepts and shows exactly how poor security practices lead to compromise. Three attack vectors are covered: physical access abuse, weak passwords and software vulnerabilities.

**OS Security Fundamentals:**

An OS is only as secure as its configuration. The OS itself provides the tools: user accounts, file permissions, updates, authentication. Whether those tools are used correctly is a human decision.

**The Three Weaknesses:**

```
+--------------------------------------------------+
|           OS ATTACK SURFACE                      |
|                                                  |
|  1. PHYSICAL ACCESS                              |
|     Boot from USB → bypass OS → access files     |
|     Mitigation: BIOS password, disk encryption   |
|                                                  |
|  2. WEAK PASSWORDS                               |
|     Default creds, dictionary words, reuse       |
|     Mitigation: Strong passwords, MFA, lockout   |
|                                                  |
|  3. UNPATCHED SOFTWARE                           |
|     Known CVEs with public exploits              |
|     Mitigation: Patch management, auto-updates   |
+--------------------------------------------------+
```

**The Lab - SSH and Manual Password Guessing:**

The scenario: you visit a client office and spot a sticky note with `sammie` and `dragon` written on it. Classic weak credential discovery.

**Step 1 - Connect as Sammie:**

```bash
ssh sammie@MACHINE_IP
# Password: dragon
# Type yes when asked about the SSH fingerprint
```

This demonstrates how physical access to a workspace can lead to credential theft. Sticky note passwords are a real and recurring problem.

**Step 2 - Find Johnny's Password:**

The room provides a list of the top 20 most common passwords. Johnny's password is somewhere in the first 7. Try them manually using `su - johnny`:

```bash
su - johnny
# Try: abc123
```

`abc123` is password number 7 on the common list. It works.

**Step 3 - Find the Root Password in History:**

Johnny typed the root password as a command by mistake. Check his command history:

```bash
history
```

The root password appears in the command history as a mistyped command.

**Step 4 - Switch to Root and Get the Flag:**

```bash
su - root
# Enter the password found in history
cat /root/flag.txt
```

**Task Answers**

| Task | Question | Answer |
|---|---|---|
| Task 1 | Which of the following is not an OS? (Thunderbird / Windows / macOS) | Thunderbird |
| Task 2 | What is an example of a password that is NOT in common password lists? | LearnM00r |
| Task 3.1 | What is the password for the user johnny? | abc123 |
| Task 3.2 | What is the root password? | (Found in johnny's command history in the lab) |
| Task 3.3 | What is the content of /root/flag.txt? | THM{YouGotRoot} |

---

## What I Learned / Reinforced

**The kernel/user space distinction is a security boundary, not just an architecture detail.** Rootkits achieve kernel-level code execution specifically because it puts them above the detection capabilities of user space AV and EDR tools. Knowing this boundary exists helps explain why certain malware is so hard to detect and why Secure Boot, measured boot and kernel integrity checking matter.

**Linux CLI is non-negotiable for this career.** At Teleperformance, most SIEM work was GUI-based. But any time I needed to investigate something deeper on a Linux host, knowing basic navigation, file searching and reading commands saved significant time. The `find` command alone is something I have used in real incident investigation to locate artefacts.

**Command history is a goldmine during investigations.** The lab demonstrates this from the attacker side but it is equally true from the analyst side. In incident response, checking `.bash_history` and PowerShell history on a compromised host often reveals attacker commands, staging directories and exfiltration attempts. I have seen this in threat hunting scenarios at work.

**Sticky note passwords are not just a training scenario.** Physical reconnaissance is a real OSINT and social engineering vector. Offensive security teams (red teamers) photograph whiteboards, sticky notes and screens during physical assessments. Educating users about not writing credentials down is a basic but important part of security awareness.

**The OS Security room is the conceptual bridge between Module 4 and real attack/defence content.** The three categories (physical access, weak passwords, unpatched software) map almost exactly to the top causes of breach in incident reports year after year. It is not a coincidence that MITRE ATT&CK techniques like Valid Accounts (T1078), Exploit Public-Facing Application (T1190) and OS Credential Dumping (T1003) appear in nearly every major breach report.

---

## Resources

- [Operating Systems: Introduction - TryHackMe](https://tryhackme.com/room/operatingsystemsintroduction)
- [Windows Basics - TryHackMe](https://tryhackme.com/room/windowsbasics)
- [Linux CLI Basics - TryHackMe](https://tryhackme.com/room/linuxclibasics)
- [Windows CLI Basics - TryHackMe](https://tryhackme.com/room/windowsclibasics)
- [Operating System Security - TryHackMe](https://tryhackme.com/room/operatingsystemsecurity)
- [MITRE ATT&CK - Valid Accounts T1078](https://attack.mitre.org/techniques/T1078/)
- [CIS Benchmarks for OS Hardening](https://www.cisecurity.org/cis-benchmarks)

---

*Written by fiza.sk293 · [GitHub](https://github.com/FizaShaikh293/thm-writeups)*
