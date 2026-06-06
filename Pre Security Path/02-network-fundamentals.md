---
layout: default
title: network fundamentals
---

**Learning Path:** Pre-Security  
**Module:** 02 - Network Fundamentals  
**Rooms Completed:** What is Networking? · Intro to LAN · OSI Model · Packets and Frames · Extending Your Network  
**Date Completed:** June 2026  
**Difficulty:** Beginner (Theory + Interactive Labs)

---

## Overview

Module 2 is the biggest content block in the Pre-Security path. Five rooms covering how networks are actually structured, how data moves across them and what protocols make it all work. This is foundational knowledge for both offensive and defensive security work. You cannot meaningfully read a SIEM alert about suspicious traffic without understanding what normal traffic looks like at the packet level. You cannot scope a pentest without knowing how routers, switches and firewalls relate to each other.

The OSI model in particular is one of those things that comes up constantly. In a SOC environment I used it implicitly every time I triaged an alert: is this a layer 3 routing issue or a layer 7 application anomaly? The model gives you a mental framework for placing the problem. This module builds that foundation properly.

---

## Room 1 - What is Networking?

**Room Link:** https://tryhackme.com/room/whatisnetworking  
**Format:** Reading + Interactive Labs

### What It Covers

Networks are devices connected together to share data. The internet is a network of networks. The room starts simple and builds up to the two ways devices identify each other: IP addresses and MAC addresses.

**IP Addresses** are logical identifiers assigned to devices. They can change depending on the network a device connects to. They are used to route data between different networks.

**MAC Addresses** are physical identifiers burned into a device's network interface card at the factory. A MAC address looks like `a4:c3:f0:85:ac:2d`, six pairs of hex values. Unlike IP addresses they do not change (though they can be spoofed, which is the point of one of the labs here).

**Ping** uses ICMP (Internet Control Message Protocol) to test connectivity between two devices. It sends a packet and waits for a reply. If the reply comes back, the connection works. The time it takes is the latency. This is one of the first tools you reach for when troubleshooting network issues.

### The Labs

**MAC Spoofing Lab:** Simulates a hotel Wi-Fi network where Alice has paid for access and Bob has not. The router is filtering by MAC address so Bob's packets get dropped. By changing Bob's MAC address to match Alice's, you bypass the filter and get access. This is a real attack class. MAC address filtering is not a strong security control for exactly this reason.

**Ping Lab:** You ping `8.8.8.8` (Google's public DNS server) from the in-browser terminal. After four pings the terminal outputs a summary with the flag.

### Task Answers

| Task | Question | Answer |
|------|----------|--------|
| Task 1 | What is the key term for devices that are connected together? | `Network` |
| Task 2 | Who invented the World Wide Web? | `Tim Berners-Lee` |
| Task 3 | What does the term "IP" stand for? | `Internet Protocol` |
| Task 3 | What is each section of an IP address called? | `Octet` |
| Task 3 | How many sections does an IP address have? | `4` |
| Task 3 | What does the term "MAC" stand for? | `Media Access Control` |
| Task 3 | Deploy the interactive lab and spoof your MAC address to access the site. What is the flag? | `THM{YOU_GOT_ON_WIFI}` |
| Task 4 | What protocol does ping use? | `ICMP` |
| Task 4 | What is the flag? | `THM{I_PINGED_THE_SERVER}` |

---

## Room 2 - Intro to LAN

**Room Link:** https://tryhackme.com/room/introtolan  
**Format:** Reading + Interactive Lab

### What It Covers

LAN stands for Local Area Network. This room covers how LANs are designed and what protocols operate within them.

**Network Topologies** are the physical or logical layouts of a network. Three main types:

- **Star Topology:** All devices connect to a central switch or hub. Most common in modern networks. Reliable and scalable but expensive. If the central device fails the whole network goes down.
- **Bus Topology:** All devices share one cable. Simple and cheap but a single break in the cable kills the whole network. Not used much anymore.
- **Ring Topology:** Devices connect in a loop, each passing data to the next. Predictable performance but a single failure breaks the ring unless there is redundancy built in.

**Subnetting** divides a network into smaller segments. This improves performance, reduces broadcast traffic and makes networks easier to manage. A subnet mask like `255.255.255.0` defines which part of an IP address identifies the network and which part identifies individual devices.

**ARP (Address Resolution Protocol)** resolves IP addresses to MAC addresses within a local network. When a device wants to communicate with another device on the same network it sends an ARP Request broadcast asking which device has a specific IP. The device with that IP sends back an ARP Reply with its MAC address. The requesting device stores this in its ARP cache for future use.

**DHCP (Dynamic Host Configuration Protocol)** automatically assigns IP addresses to devices on a network. The process is:

1. **Discover** - Device broadcasts looking for a DHCP server
2. **Offer** - DHCP server offers an available IP address
3. **Request** - Device requests the offered IP
4. **Acknowledge** - DHCP server confirms the assignment

The acronym to remember this is DORA.

### The Lab

An interactive topology lab where you break different network layouts to understand their failure modes. Getting through it correctly produces the flag.

**Flag:** `THM{TOPOLOGY_FLAWS}`

### Task Answers

| Task | Question | Answer |
|------|----------|--------|
| Task 1 | What does LAN stand for? | `Local Area Network` |
| Task 1 | What is the technical term for the design or look of a network? | `Topology` |
| Task 1 | What device is used to centrally connect multiple devices on a local network? | `Switch` |
| Task 1 | What topology is pictured? (Star) | `Star Topology` |
| Task 1 | Complete the interactive lab. What is the flag? | `THM{TOPOLOGY_FLAWS}` |
| Task 2 | What is the technical term for dividing a network into smaller pieces? | `Subnetting` |
| Task 2 | How many bits make up a subnet mask? | `32` |
| Task 2 | What is the name used to identify the device responsible for sending data to another network? | `Default Gateway` |
| Task 3 | What does ARP stand for? | `Address Resolution Protocol` |
| Task 3 | What category of ARP packet asks a device whether or not it has a specific IP address? | `Request` |
| Task 3 | What address is used as a physical identifier for a device on a network? | `MAC Address` |
| Task 3 | What address is used as a logical identifier for a device on a network? | `IP Address` |
| Task 4 | What type of DHCP packet is used by a device to retrieve an IP address? | `Discover` |
| Task 4 | What type of DHCP packet does a device send once it has been offered an IP address? | `Request` |
| Task 4 | Finally, what does DHCP stand for? | `Dynamic Host Configuration Protocol` |

---

## Room 3 - OSI Model

**Room Link:** https://tryhackme.com/room/osimodelzi  
**Format:** Reading + Game

### What It Covers

The OSI model (Open Systems Interconnection) is a seven-layer framework that standardises how data is transmitted between devices. Every networking concept you encounter maps onto one of these layers. Understanding it makes it much easier to diagnose problems and understand where an attack is happening.

The layers from top to bottom:

| Layer | Name | What It Does |
|-------|------|-------------|
| 7 | Application | Where user-facing applications operate. HTTP, FTP, DNS, email clients |
| 6 | Presentation | Data formatting, encryption, compression. Translates between application and network formats |
| 5 | Session | Manages and maintains connections between devices. Handles authentication and reconnection |
| 4 | Transport | Controls how much data is sent at once and ensures reliable delivery. TCP and UDP live here |
| 3 | Network | Handles logical addressing (IP) and routing between different networks |
| 2 | Data Link | Handles physical addressing (MAC) and formats data for transmission on the local network |
| 1 | Physical | The actual cables, signals and hardware. Converts data to electrical, optical or radio signals |

**Encapsulation** is the process of each layer adding its own header information to the data as it travels down the stack on the sending side. On the receiving side the process reverses (decapsulation) and each layer strips off its header.

In a SOC context I use this model constantly without thinking about it. When an alert fires on port 443 traffic from an unexpected internal host, that is a layer 4 (Transport) and layer 7 (Application) investigation. When a switch is dropping packets that is a layer 2 or layer 1 problem. The model gives you a starting point for where to look.

### The Lab

A game where you place attacks and network events into the correct OSI layer. Completing it gives you the flag.

**Flag:** `THM{OSI_DUNGEON_ESCAPED}`

### Task Answers

| Task | Question | Answer |
|------|----------|--------|
| Task 1 | What does OSI stand for? | `Open Systems Interconnection` |
| Task 1 | How many layers does the OSI model have? | `7` |
| Task 1 | What is the key term for when pieces of information get added to data? | `Encapsulation` |
| Task 2 | What is the name of this layer? (Layer 7) | `Application` |
| Task 2 | What is the technical term for the software that users interact with? | `Graphical User Interface` |
| Task 3 | What is the name of this layer? (Layer 6) | `Presentation` |
| Task 3 | What is the main purpose of the presentation layer? | `Translation` |
| Task 4 | What is the name of this layer? (Layer 5) | `Session` |
| Task 4 | What is the technical term for when a connection is established? | `Session` |
| Task 5 | What is the name of this layer? (Layer 4) | `Transport` |
| Task 5 | What does UDP stand for? | `User Datagram Protocol` |
| Task 5 | What does TCP stand for? | `Transmission Control Protocol` |
| Task 5 | What type of connection is reliable and ensures data is received? | `TCP` |
| Task 6 | What is the name of this layer? (Layer 3) | `Network` |
| Task 6 | Will packets take the most optimal route across a network? | `Y` |
| Task 7 | What is the name of this layer? (Layer 2) | `Data Link` |
| Task 7 | What is the name of this addressing system used at this layer? | `MAC Address` |
| Task 8 | What is the name of this layer? (Layer 1) | `Physical` |
| Task 9 | Complete the game. What is the flag? | `THM{OSI_DUNGEON_ESCAPED}` |

---

## Room 4 - Packets and Frames

**Room Link:** https://tryhackme.com/room/packetsframes  
**Format:** Reading + Interactive Lab

### What It Covers

Data does not travel across networks as one large blob. It gets broken into smaller pieces called **packets** at layer 3 (with IP addressing information) and **frames** at layer 2 (without IP addressing, just MAC addresses for the local hop). Think of a packet as an envelope with a full address on it, and a frame as the inner envelope that only has local delivery instructions.

**TCP (Transmission Control Protocol)** is connection-oriented. Before any data is sent, a three-way handshake happens:

1. **SYN** - Client sends a synchronise packet to initiate the connection
2. **SYN/ACK** - Server acknowledges and sends its own synchronise
3. **ACK** - Client acknowledges the server's response

Only after this is complete does data start flowing. TCP also handles error checking and retransmission of lost packets. This reliability comes at the cost of speed.

**UDP (User Datagram Protocol)** is connectionless. It fires packets and does not check whether they arrived. Much faster than TCP but no guarantee of delivery. Used for things like video streaming, online gaming and DNS lookups where speed matters more than perfect reliability.

**Ports** are how a single device runs multiple services simultaneously. Port numbers from 0 to 65535. Common ones worth knowing:

| Port | Protocol |
|------|----------|
| 21 | FTP |
| 22 | SSH |
| 80 | HTTP |
| 443 | HTTPS |
| 3389 | RDP |

Understanding ports is essential for reading nmap output. Every open port in a scan result is an attack surface.

### Task Answers

| Task | Question | Answer |
|------|----------|--------|
| Task 1 | What is the name for a piece of data when it does have IP addressing information? | `Packet` |
| Task 1 | What is the name for a piece of data when it does not have IP addressing information? | `Frame` |
| Task 2 | What is the header in a TCP packet that ensures the integrity of data? | `Checksum` |
| Task 2 | Provide the order of a connection: | `SYN, SYN/ACK, ACK` |
| Task 3 | What is the technical term given to the process of connection termination? | `Closing` |
| Task 3 | What type of connection is NOT reliable? | `UDP` |
| Task 4 | What is the flag shown on the interactive lab? | `THM{TELNET_MASTER}` |

---

## Room 5 - Extending Your Network

**Room Link:** https://tryhackme.com/room/extendingyournetwork  
**Format:** Reading + Interactive Lab

### What It Covers

This room covers the devices and techniques that connect networks together and control what traffic is allowed through.

**Port Forwarding** allows services inside a private network to be accessible from the internet. If you run a web server at home on port 80, your router needs a port forwarding rule that says: any incoming traffic on port 80 from the internet goes to this internal IP. Without it the router has no idea which internal device should receive the connection.

**Firewalls** filter network traffic based on rules. Two main types:

- **Stateful Firewalls** track the state of active connections. They understand context. If a connection was established legitimately from the inside, related return traffic is allowed. More intelligent and resource-intensive.
- **Stateless Firewalls** apply fixed rules to each packet independently with no context of what came before. Simpler, faster but easier to bypass.

**VPNs (Virtual Private Networks)** create encrypted tunnels between devices or networks. A VPN makes remote devices appear as if they are on the local network. Used legitimately for remote work and privacy but also commonly used by attackers to tunnel traffic and avoid detection. Seeing unexpected VPN traffic in SIEM logs is always worth investigating.

**Routers** direct traffic between different networks. They operate at layer 3 (network layer) and use routing tables to decide where to send packets.

**Switches** connect multiple devices within the same network. They operate at layer 2 (data link layer) and use MAC address tables to send frames only to the correct destination port rather than broadcasting to everyone like a hub would.

### The Lab

An interactive lab where you configure a network correctly including port forwarding rules to get services accessible from outside. Completing it gives the flag.

**Flag:** `THM{EXTENDING_YOUR_NETWORK}`

### Task Answers

| Task | Question | Answer |
|------|----------|--------|
| Task 1 | What is the name of the device that is used to configure port forwarding? | `Router` |
| Task 1 | What is the name of the device that is used to connect multiple devices? | `Switch` |
| Task 2 | What is the name of the device used to inspect and filter traffic? | `Firewall` |
| Task 2 | What type of firewall inspects the entire connection? | `Stateful` |
| Task 2 | What type of firewall inspects each packet? | `Stateless` |
| Task 3 | What VPN technology only encrypts and provides the authentication of data? | `PPP` |
| Task 3 | What VPN technology uses the IP framework? | `IPSec` |
| Task 4 | What is the flag from the interactive lab? | `THM{EXTENDING_YOUR_NETWORK}` |

---

## What I Learned / Reinforced

**The OSI model is a diagnostic tool not just exam content.** Every time you look at a network problem you are implicitly working through the layers. Knowing the model explicitly makes that process faster. Layer 1 is physical, check the cable. Layer 2 is MAC addressing, check ARP tables. Layer 3 is IP routing. Layer 4 is TCP/UDP. Layer 7 is the application. Start at the bottom and work up.

**MAC addresses can be spoofed.** The hotel Wi-Fi lab drives this home immediately. MAC address filtering is not a security control worth relying on. I have seen this referenced in vulnerability reports from clients who thought it was sufficient access control for their guest networks.

**TCP vs UDP is about reliability vs speed.** This distinction matters a lot in security. TCP leaves a clean three-way handshake in logs. UDP is noisier and harder to track. DNS exfiltration often uses UDP for exactly this reason. Knowing which protocols use which transport helps you understand what you are looking at in packet captures.

**Stateful firewalls are significantly stronger than stateless ones.** A stateless firewall only knows the rule. A stateful firewall knows whether this packet makes sense given the conversation that preceded it. This is why understanding context in SIEM monitoring matters so much, a packet in isolation tells you less than a packet in the context of a session.

**ARP and DHCP are both attack surfaces.** ARP has no authentication. ARP poisoning (sending fake ARP replies to map your MAC to another device's IP) is a real attack that enables man-in-the-middle interception. DHCP starvation attacks flood a server with requests to exhaust the available IP pool. Both are worth understanding from both attack and detection perspectives.

---

## Resources

- [What is Networking - TryHackMe](https://tryhackme.com/room/whatisnetworking)
- [Intro to LAN - TryHackMe](https://tryhackme.com/room/introtolan)
- [OSI Model - TryHackMe](https://tryhackme.com/room/osimodelzi)
- [Packets and Frames - TryHackMe](https://tryhackme.com/room/packetsframes)
- [Extending Your Network - TryHackMe](https://tryhackme.com/room/extendingyournetwork)
- [Wireshark - Packet Analysis Tool](https://www.wireshark.org/)
- [TCP/IP vs OSI Model - Cloudflare](https://www.cloudflare.com/learning/ddos/glossary/open-systems-interconnection-model-osi/)

---

*Written by fiza.sk293 · [GitHub](https://github.com/FizaShaikh293/thm-writeups)*
