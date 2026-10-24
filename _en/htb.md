---
title: "Hack The Box — Writeups"
date: 2026-10-24 20:00:00 +0200
categories: [HTB]
tags: [hackthebox, htb, oscp, pentesting]
pin: true
ref: htb-index
description: "Writeups of retired Hack The Box machines, organized into Linux, Windows and Active Directory. OSCP preparation following the NetSecFocus Trophy Room (TJNull's list)."
toc: false
---

In this section I document my **[Hack The Box](https://www.hackthebox.com/) machine writeups**. I solve each box myself first and then write up the whole process: I don't stop at the command that works, I explain **why** it works and, whenever there's more than one path, I also show the **alternative routes**.

It's part of my **OSCP** preparation, following the recommended machines from the [NetSecFocus Trophy Room](https://docs.google.com/spreadsheets/d/1dwSMIAPIam0PuRBkCiDI88pU3yzrqqHkDtBngUHNCw8/htmlview) (the well-known **TJNull** list).

> I only publish writeups for **retired machines**, following Hack The Box's rules. Flags (`user.txt` and `root.txt`) are always **redacted**: the point is to learn the technique, not hand out the answer.
{: .prompt-warning }

## How it's organized

Each machine is a standalone writeup with the same structure: **recon → enumeration → initial access (user) → privilege escalation (root) → other paths → mitigation**. I group them into three blocks:

### 🐧 Linux

Service enumeration, web exploitation, SUID binary abuse, cron jobs, containers and *nix privilege escalation.

➡️ [See Linux machines](/categories/linux/)

### 🪟 Windows

SMB/RPC enumeration, privilege abuse (`SeImpersonate` and friends), misconfigured services and local escalation.

➡️ <a href="/categories/windows/" data-proofer-ignore>See Windows machines</a>

### 🏛️ Active Directory

Attacks against Windows domains: Kerberoasting, AS-REP roasting, ACL abuse, lateral movement and domain takeover. The key block for the OSCP.

➡️ <a href="/categories/active-directory/" data-proofer-ignore>See Active Directory machines</a>

---

The list grows as I finish machines. If this is your first time here, take a look at the [Bandit](/categories/bandit/) series too, where I start from scratch with the Linux terminal.
