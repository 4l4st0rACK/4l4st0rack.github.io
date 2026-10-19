---
title: "OverTheWire Bandit: Level 19 → 20 Walkthrough"
date: 2026-10-19 20:44:00 +0200
categories: [Linux, Bandit]
tags: [bandit, overthewire, linux, suid, permissions, privesc]
description: OverTheWire Bandit level 19 to 20 walkthrough, using a SUID binary to run commands as another user, a hands-on introduction to privilege escalation.
ref: bandit-19-20
serie: bandit
nivel: 20
---

This level is a first taste of **privilege escalation**, one of the pillars of pentesting. The idea: a specific program can run with the permissions of **another user** thanks to a special bit called **SUID**. We're going to use that to read a file we shouldn't be able to read.

> Level passwords are shown as **[REDACTED]**. Following OverTheWire's rules, I don't publish credentials: the fun part is getting there yourself.
{: .prompt-warning }

## Connecting

```console
$ sshpass -p '[REDACTED]' ssh bandit19@bandit.labs.overthewire.org -p 2220
bandit19@bandit:~$ export TERM=xterm
```

## The challenge

The [level goal](https://overthewire.org/wargames/bandit/bandit20.html) says that to reach the next level you have to use a **setuid binary** in the home directory. It asks us to run it without arguments to see how it's used, and adds a hint: where each level's password is usually stored (yes, `/etc/bandit_pass/`).

```console
bandit19@bandit:~$ ls
bandit20-do
```

A single file: `bandit20-do`. Let's look at its permissions:

```console
bandit19@bandit:~$ ls -l bandit20-do
-rwsr-x--- 1 bandit20 bandit19 ... bandit20-do
```

Here's the key, and you need to know how to read it:

- The file's owner is **bandit20**, and the group is **bandit19** (us).
- In the owner's permissions, where there would normally be an `x` (`-rwx`), there's an **`s`**: `-rws`. That `s` is the **SUID bit**.
- Our group (`bandit19`) has `r-x`: we can read and **execute** the binary.

## What does the SUID bit do?

Normally, when you run a program, it runs **with your permissions**. The SUID bit (Set User ID) changes that: it makes the program run **with its owner's permissions**, not those of whoever launches it.

Since `bandit20-do` has SUID set and belongs to bandit20, running it means we **act as bandit20** for as long as that program runs. And bandit20 can read `/etc/bandit_pass/bandit20`, even though we (bandit19) can't.

First I run it without arguments, as the goal asks, to see what it does:

```console
bandit19@bandit:~$ ./bandit20-do
Run a command as another user.
  Example: ./bandit20-do id
```

It's a "launcher": you pass it a command and it runs it as bandit20. We can confirm with `id`:

```console
bandit19@bandit:~$ ./bandit20-do id
uid=11019(bandit19) gid=11019(bandit19) euid=11020(bandit20) ...
```

Notice the `euid=11020(bandit20)`: the **euid** (effective user ID) is bandit20. That's exactly what we want: for permission purposes, we are bandit20.

## The solution

I ask the binary to read the password as bandit20:

```console
bandit19@bandit:~$ ./bandit20-do cat /etc/bandit_pass/bandit20
[REDACTED]
```

And there it is. The `cat` runs as bandit20, who does have permission to read that file.

> The `./` in front of the name is mandatory. The current directory isn't in `PATH` for security reasons, so to run a program in the folder you're in you have to say so explicitly with `./`.
{: .prompt-info }

## Why this matters in security

SUID binaries are one of the first things you look for when auditing a Linux machine, because a badly made SUID (or one that shouldn't exist) is a direct path to **privilege escalation**, sometimes all the way to `root`. Here the binary is well scoped (it only acts as a launcher for bandit20), but the concept is the same one you'll exploit on real machines.

To find all the SUID binaries on a system you use `find`:

```console
bandit19@bandit:~$ find / -perm -4000 -type f 2>/dev/null
```

- `-perm -4000` looks for files that have the SUID bit set (4000 in octal).
- `-type f` limits the search to files.
- `2>/dev/null` discards the "permission denied" errors from folders we can't read.

> This is, literally, one of the first checks you'll run on any HTB machine or on the OSCP when looking to escalate privileges. Write that command down.
{: .prompt-tip }

## Moving to the next level

```console
bandit19@bandit:~$ exit
$ sshpass -p '[REDACTED]' ssh bandit20@bandit.labs.overthewire.org -p 2220
```

## Summary

| Command / concept | What it's for |
| --- | --- |
| SUID bit (`-rws`) | Makes a program run with its owner's permissions |
| `ls -l file` | See the permissions and spot the SUID `s` |
| `./bandit20-do command` | Run a command as the binary's owner (bandit20) |
| `id` / `euid` | Check which identity a process is running as |
| `./` | Run a binary from the current directory (not in PATH) |
| `find / -perm -4000 -type f 2>/dev/null` | Find all SUID binaries on the system |

{% include series-nav.html %}
