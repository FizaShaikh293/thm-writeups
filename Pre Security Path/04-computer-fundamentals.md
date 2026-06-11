---
layout: default
title: Computer Fundamentals
---

**Learning Path:** Pre-Security  
**Module:** 04 - Computer Fundamentals  
**Rooms Completed:** Inside a Computer System · Computer Types · Client-Server Basics · Virtualisation Basics · Cloud Computing Fundamentals  
**Date Completed:** April - May 2026  
**Difficulty:** Beginner (Theory + Interactive Labs)

---

## Overview

Module 4 covers the physical and logical foundations of computing: what is inside a machine, the different forms computers take, how clients and servers communicate, how virtualisation works and how cloud infrastructure is built on top of all of it. This module is less glamorous than the attack and defence content but it is genuinely foundational. Security vulnerabilities do not exist in the abstract. They exist in hardware, in operating systems, in virtualised environments and in cloud configurations. Understanding what you are securing is the prerequisite for securing it.

From an incident response perspective this module is directly relevant. When investigating a compromised system you need to understand the boot process to know what malware can do before the OS loads. When reviewing a cloud environment alert you need to understand the service model to know who is responsible for what.

---

## Room 1 - Inside a Computer System

**Room Link:** https://tryhackme.com/room/insideacomputer  
**Format:** Reading + Component Placement Lab

### What It Covers

Every computer, regardless of form factor, is built from the same core components. The room uses a human body analogy to make each one memorable.

**Core Components:**

| Component | Analogy | Function |
|-----------|---------|----------|
| CPU (Central Processing Unit) | The Brain | Executes instructions and performs calculations |
| RAM (Random Access Memory) | Short-term Memory | Temporary, fast storage for data the CPU is actively using |
| HDD / SSD (Storage) | Long-term Memory | Persistent storage for the OS, files and applications |
| Motherboard | The Skeleton and Nerves | The main circuit board connecting all components |
| PSU (Power Supply Unit) | Heart and Lungs | Converts mains power to the voltages the components need |
| GPU (Graphics Processing Unit) | Visual Cortex | Handles rendering and graphical output |

**The Boot Process:**

Understanding how a computer starts up matters in security because the early boot stages run before the operating system loads and are therefore before most security software is active. Bootkits and firmware malware exploit exactly this window.

1. Power is applied and the **UEFI/BIOS** firmware activates
2. **POST (Power-On Self Test)** runs to check all required hardware is present and functional
3. UEFI/BIOS locates the configured boot device (SSD, USB, network)
4. The **Bootloader** (e.g. GRUB on Linux, Windows Boot Manager) is loaded from the boot device
5. The Bootloader loads the **Operating System** kernel into RAM
6. The OS initialises drivers, services and user space

The distinction between BIOS and UEFI is worth knowing. BIOS is the legacy firmware standard. UEFI is the modern replacement with support for larger drives, faster boot times and Secure Boot, which cryptographically verifies each stage of the boot process to prevent tampering.

### Screenshot

**Component Placement Lab - Flag THM{4llpccomp0n3nts1d3nt1f13d}**

The interactive lab asks you to drag each component to its correct connector on a motherboard diagram. Getting all placements correct reveals the flag.

![Inside a Computer component placement lab showing flag THM{4llpccomp0n3nts1d3nt1f13d}](../screenshots/Screenshot_2026-04-29_001016.png)

### Task Answers

| Task | Question | Answer |
|------|----------|--------|
| Task 1 | No answer needed | - |
| Task 2 | What is the component that processes instructions and performs calculations? | `CPU` |
| Task 2 | What is the name of the circuit board that connects all components? | `Motherboard` |
| Task 2 | What does RAM stand for? | `Random Access Memory` |
| Task 2 | What type of storage retains data even after the computer is turned off? | `SSD/HDD` |
| Task 3 | What is the name of the firmware that initialises hardware before the OS loads? | `UEFI` |
| Task 3 | What does POST stand for? | `Power-On Self Test` |
| Task 3 | What is the flag from the boot process exercise? | `THM{4llpccomp0n3nts1d3nt1f13d}` |

---

## Room 2 - Computer Types

**Room Link:** https://tryhackme.com/room/computertypes  
**Format:** Interactive Guided Story Lab (Sophia at Nova Labs)

### What It Covers

Computers are not all desktops and laptops. This room covers the full range of device categories and the security implications of each.

**Device Categories:**

| Type | Description | Examples |
|------|-------------|---------|
| Desktop | Fixed workstation, modular, upgradeable | Office PCs, gaming rigs |
| Laptop | Portable, integrated components, battery powered | Work laptops, notebooks |
| Server | High uptime, rack-mounted or tower, handles many simultaneous requests | Web servers, database servers, domain controllers |
| Workstation | High-performance desktop for specialist tasks | CAD, video editing, scientific computing |
| Embedded System | Purpose-built hardware running fixed software | Routers, industrial controllers, medical devices |
| IoT (Internet of Things) | Connected devices, often with minimal security | Smart TVs, thermostats, cameras, smart speakers |
| Smartphone / Tablet | Mobile general-purpose computers | iOS and Android devices |

**Why this matters for security:**

IoT and embedded devices are a major and growing attack surface. They often run outdated firmware, have default credentials that are never changed and are not covered by standard endpoint security tooling. Compromised IoT devices are a common entry point for network intrusion and are frequently recruited into botnets. Understanding that a smart plug or CCTV camera is a computer with an IP address and network stack is the starting point for securing them.

### Screenshot

**Computer Types Quiz - Flag THM{8_computer_types}**

The final task is a quiz inside the Nova Labs simulation. Scoring at least 4 out of 5 completes the training and reveals the flag.

![Computer Types final quiz showing 5/5 correct and flag THM{8_computer_types}](../screenshots/Screenshot_2026-05-06_204658.png)

### Task Answers

| Task | Question | Answer |
|------|----------|--------|
| Task 1 | No answer needed | - |
| Task 2 | What type of computer is designed to handle requests from multiple clients simultaneously? | `Server` |
| Task 3 | What type of system is built into a device for a specific purpose? | `Embedded System` |
| Task 4 | What does IoT stand for? | `Internet of Things` |
| Task 5 | Complete the quiz. What is the flag? | `THM{8_computer_types}` |

---

## Room 3 - Client-Server Basics

**Room Link:** https://tryhackme.com/room/clientserverbasics  
**Format:** Reading (Theory)

### What It Covers

The client-server model is the fundamental architecture of networked computing. A client requests a service or resource. A server provides it. Every web request, DNS lookup, database query and API call follows this pattern.

**Clients** are devices or software that initiate requests. Your browser is a client. A mobile app is a client. The THM AttackBox terminal is a client when it connects to a target machine.

**Servers** listen on specific ports for incoming connections and respond to requests. A web server listens on port 80 or 443. An SSH server listens on port 22. A DNS server listens on port 53.

**Protocols** are the agreed rules for how clients and servers communicate. HTTP defines how a browser and web server exchange data. FTP defines how files are transferred. SMTP defines how email is sent. Both sides must speak the same protocol for communication to work.

**Ports** allow a single machine to run multiple services simultaneously. When a packet arrives at a server, the port number tells the OS which application should handle it. Port numbers 0-1023 are well-known ports reserved for standard services. 1024-49151 are registered ports. 49152-65535 are dynamic/private ports.

Understanding the client-server model is foundational for both pentesting and monitoring. In a pentest you are typically a client attacking a server. In a SOC you are watching for clients (internal or external) behaving abnormally when communicating with servers.

### Task Answers

| Task | Question | Answer |
|------|----------|--------|
| Task 1 | What is the name of the model where one device requests services from another? | `Client-Server` |
| Task 2 | What is the device that provides resources or services called? | `Server` |
| Task 2 | What is the device that requests resources or services called? | `Client` |
| Task 3 | What are the rules that define how clients and servers communicate called? | `Protocols` |
| Task 3 | What port does HTTP use by default? | `80` |
| Task 3 | What port does HTTPS use by default? | `443` |
| Task 3 | What port does SSH use by default? | `22` |

---

## Room 4 - Virtualisation Basics

**Room Link:** https://tryhackme.com/room/virtualisationbasics  
**Format:** Reading + Scenario Questions

### What It Covers

Virtualisation allows a single physical machine to run multiple independent operating systems simultaneously. Each virtual machine (VM) is isolated from the others and behaves as if it has its own dedicated hardware even though they all share the same physical CPU, RAM and storage.

**The Hypervisor** is the software layer that makes this possible. It manages and allocates physical resources across VMs and enforces isolation between them.

Two types:

- **Type 1 Hypervisor (Bare Metal):** Runs directly on the physical hardware with no host OS underneath. Used in enterprise data centres and cloud providers. Examples: VMware ESXi, Microsoft Hyper-V, Xen. Better performance, more secure.
- **Type 2 Hypervisor (Hosted):** Runs as an application on top of a standard OS. Used on personal machines for home labs and testing. Examples: VirtualBox, VMware Workstation. Easier to set up, slightly less performance.

**Containers** are a lighter alternative to VMs. Rather than virtualising the entire hardware stack with a separate OS, containers share the host OS kernel and only package the application and its dependencies. Much faster to spin up, much lower overhead. Docker is the dominant container platform. Kubernetes manages containers at scale.

| | Virtual Machine | Container |
|---|---|---|
| Isolation | Full OS separation | Shared kernel |
| Boot time | Minutes | Seconds |
| Size | GBs | MBs |
| Use case | Full environment isolation | Application deployment |

**Security relevance:** VMs are used extensively in security work. Malware analysis is done inside isolated VMs so that when a sample executes it cannot affect the analyst's real machine. THM itself runs the target machines you attack as VMs. Understanding snapshots (saved VM states you can roll back to) is useful for both lab work and incident response.

### Task Answers

| Task | Question | Answer |
|------|----------|--------|
| Task 1 | What does virtualisation enable multiple applications to share? | `Physical Hardware` |
| Task 2 | What is the name of the software that manages resources for each virtual machine? | `Hypervisor` |
| Task 2 | A user wants to deploy a study lab on their personal machine. What type of hypervisor will they use? | `Type 2` |
| Task 2 | A company wants to host multiple small applications in the same VM. What should they use? | `Containers` |
| Task 3 | No answer needed | - |

---

## Room 5 - Cloud Computing Fundamentals

**Room Link:** https://tryhackme.com/room/cloudcomputingfundamentals  
**Format:** Reading + Cloud Instance Cost Lab

### What It Covers

Cloud computing is the delivery of computing resources over the internet on a pay-as-you-go basis. Instead of owning and maintaining physical servers, organisations rent infrastructure, platforms or software from cloud providers. The three major providers are AWS, Microsoft Azure and Google Cloud Platform.

**Key Characteristics:**

- **Scalability:** Resources can be increased or decreased based on demand automatically. This is the core advantage over physical infrastructure.
- **On-demand:** Resources are provisioned when needed and released when not.
- **Broad access:** Services are accessible from anywhere with internet connectivity.

**Cloud Deployment Models:**

| Model | Description |
|-------|-------------|
| Public Cloud | Infrastructure owned and operated by a third-party provider, shared across customers |
| Private Cloud | Infrastructure dedicated to a single organisation, on-premises or hosted |
| Hybrid Cloud | Combination of public and private, with data and applications moving between them |

**Cloud Service Models:**

| Model | What You Manage | What Provider Manages | Example |
|-------|----------------|----------------------|---------|
| IaaS (Infrastructure as a Service) | OS, applications, data | Hardware, networking, virtualisation | AWS EC2, Azure VMs |
| PaaS (Platform as a Service) | Applications, data | Everything below the application | Heroku, Google App Engine |
| SaaS (Software as a Service) | Nothing (just use it) | Everything | Gmail, Salesforce, Office 365 |

The shared responsibility model is critical for cloud security. In IaaS the provider secures the physical infrastructure but you are responsible for securing the OS, applications and data. Misconfigurations at the customer level are the leading cause of cloud security incidents. Misconfigured S3 buckets exposing sensitive data is a famous recurring example.

### The Lab - Cloud Instance Cost Simulation

An interactive lab where you manage cloud instances (similar to AWS EC2) and calculate running costs based on which instances are active. Stopping instances reduces cost. The questions test whether you understand the pay-as-you-go billing model.

### Task Answers

| Task | Question | Answer |
|------|----------|--------|
| Task 1 | What characteristic enables handling unexpected increases in traffic? | `Scalability` |
| Task 2 | What cloud deployment model is shared across many customers? | `Public Cloud` |
| Task 2 | What model gives a single organisation dedicated infrastructure? | `Private Cloud` |
| Task 3 | You want to deploy an app focusing only on development, leaving infrastructure to others. What service model is best? | `PaaS` |
| Task 3 | You need full control over the OS and applications. What service model is best? | `IaaS` |
| Task 3 | What service model requires no management from the user? | `SaaS` |
| Task 4 | What is the total cost if study-machine-1 and study-machine-2 are stopped? | `30` |
| Task 4 | How many credits does an m5.large EC2 instance cost per month? | `70` |

---

## What I Learned / Reinforced

**The boot process is a real attack surface.** UEFI rootkits and bootkits are a real malware category. They persist below the OS level which means reinstalling the OS does not remove them. Secure Boot exists specifically to counter this. Understanding the boot sequence makes it clear why Secure Boot matters and why disabling it is a security risk.

**IoT is the weakest link in most networks.** Most enterprise security tooling covers laptops, servers and phones. The smart printer, the IP camera, the HVAC controller sitting on the same network segment are often completely unmonitored. Attackers know this. IoT compromise for network pivoting is well documented. Asset inventory that includes IoT devices is non-negotiable.

**Type 1 vs Type 2 hypervisors matters for your home lab.** When setting up TryHackMe labs locally, VirtualBox (Type 2) is the standard choice because it runs on top of your existing OS. Understanding why enterprise environments use Type 1 (bare metal, more efficient, better isolation) helps you understand cloud infrastructure.

**VMs are essential for malware analysis.** The isolation guarantee is what makes VM-based sandboxing safe. Malware cannot escape the VM and affect the host (with some exceptions for VM escape vulnerabilities, which are themselves a fascinating attack class). Every malware analyst works in VMs by default.

**Cloud misconfiguration is a bigger risk than cloud hacking.** Most high-profile cloud breaches are not sophisticated attacks. They are misconfigured storage buckets, overly permissive IAM policies and forgotten publicly exposed services. The shared responsibility model means customers are responsible for their configuration. Understanding IaaS vs PaaS vs SaaS is the starting point for understanding where those responsibilities fall.

---

## Resources

- [Inside a Computer System - TryHackMe](https://tryhackme.com/room/insideacomputer)
- [Computer Types - TryHackMe](https://tryhackme.com/room/computertypes)
- [Virtualisation Basics - TryHackMe](https://tryhackme.com/room/virtualisationbasics)
- [Cloud Computing Fundamentals - TryHackMe](https://tryhackme.com/room/cloudcomputingfundamentals)
- [AWS Shared Responsibility Model](https://aws.amazon.com/compliance/shared-responsibility-model/)
- [VirtualBox Download](https://www.virtualbox.org/)

---

*Written by fiza.sk293 · [GitHub](https://github.com/FizaShaikh293/thm-writeups)*
