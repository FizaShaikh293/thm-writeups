---
layout: default
title: AI Threat Modelling Assessment
---

# AI Threat Modelling Assessment

**Difficulty:** Easy  
**Category:** AI Security, Threat Modelling  
**Date:** 22-05-2026  
**Room:** [TryHackMe](https://tryhackme.com/room/aithreatmodellingassessment)

---

## Overview

This room puts your AI threat modelling knowledge to the test through an interactive assessment application. It builds directly on the two preceding modules covering what AI and machine learning are, how they work, and how AI systems can be assessed as attack surfaces using a structured threat modelling methodology.

Unlike most rooms, there's no machine to exploit here. The entire challenge is conceptual: you're given a simulated AI system and asked to identify threats and apply the right defences in the right places.

---

## What the Room Covers

The assessment is structured around Phase 2 — Attack Simulation. You're shown an AI pipeline — with components like User Input, API Gateway, LLM Agent, Prompt, Retrieval, and Database — and asked to think about where attacks can enter, what they target, and how to defend against them.

The two main attack scenarios in the assessment are:

- **Data Poisoning** — corrupting the inputs or data that an AI system learns from or retrieves
- **Sensitive Data Leakage** — exploiting components that handle confidential information so it surfaces in the model's output

---

## Approach

### Scenario 1: Data Poisoning

The task here is to place shields (defensive controls) on the components most at risk from data poisoning.

![Data Poisoning scenario — placing shields on components in the AI pipeline](data-poisoning.png)

When I looked at the pipeline, I thought about where poisoned data could enter or persist:

- The **Database** is the obvious target — if the embeddings or records stored there are tampered with, every query pulls back poisoned context.
- The **Retrieval** component fetches that context. If filtering is weak or the data it pulls from is compromised, there's no barrier before it reaches the model.

So I placed my two shields on **Retrieval** and **Database**. The reasoning: poisoned data has to live somewhere and get fetched somehow. Protecting those two components means you're stopping the attack at the source and at the gate.

### Scenario 2: Sensitive Data Leakage

This scenario flips the direction. Instead of corrupting what goes in, the attacker is trying to extract what comes out — getting the model to surface confidential data through its response.

![Attack Prevented — shields successfully blocked data leakage through the response chain](attack-prevented.png)

The key insight the room gives you is in the "Why It Works" breakdown:

- **LLM** — the model decides what goes into the response. Without safeguards, it may include sensitive retrieved data.
- **Retrieval** — fetches contextual data. Weak filtering means sensitive information can enter the pipeline.
- **Database** — stores the embeddings and records. If those contain confidential data and retrieval is unguarded, leakage becomes an indirect path.

Shielding the right components blocks the chain before the sensitive data can make it into the response.

---

## What I Learned

This room made me think differently about AI security compared to traditional pentesting. With a normal web app, you're looking for input validation failures, broken auth, or exposed endpoints. With an AI system, the attack surface is more distributed — it runs across a pipeline of components, and the vulnerability might not be in any single component but in how they connect.

A few things that stuck with me:

**Threat modelling requires understanding data flow.** You can't protect an AI system without knowing how data moves through it — from user input, through the LLM and retrieval components, to the database and back. Each handoff is a potential attack point.

**Defences need to match the attack direction.** Data poisoning goes *in*, so you protect ingestion and storage. Data leakage goes *out*, so you protect retrieval and generation. Getting that direction wrong means your shields are in the wrong place.

**The LLM itself is part of the attack surface.** This was the biggest mental shift. The model isn't just a tool — it's a component that can be exploited if it's fed bad data or given insufficient output controls.

---

## Room Completion

Room completed at 100%. The assessment covered everything introduced in the two AI threat modelling modules and gave a hands-on way to test whether the concepts had actually landed.

---

*More writeups added as I work through rooms. Back to [all writeups](../).*
