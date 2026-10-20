---
title: "OverTheWire Bandit: Level 26 → 27 Walkthrough"
date: 2026-10-20 21:18:00 +0200
categories: [Linux, Bandit]
tags: [bandit, overthewire, linux, suid, privesc, shell]
description: OverTheWire Bandit level 26 to 27 walkthrough, using a SUID binary to run commands as bandit27, starting from the shell obtained by escaping more.
ref: bandit-26-27
serie: bandit
nivel: 27
---

This level is the natural continuation of the previous one. We've already escaped bandit26's restricted shell, and now we run into an old friend: a **SUID binary** that executes commands as another user. If you did level 19 → 20, this will feel very familiar.

> Level passwords are shown as **[REDACTED]**. Following OverTheWire's rules, I don't publish credentials: the fun part is getting there yourself.
{: .prompt-warning }

## Starting point

We come from the [previous level](/en/posts/bandit-level-25-26/), where we escaped bandit26's restricted shell by opening `vim` from `more` and launching a shell with `:set shell=/bin/bash` and `:shell`. So we're already in as bandit26 with a normal bash:

```console
bandit26@bandit:~$ whoami
bandit26
bandit26@bandit:~$ ls
bandit27-do  text.txt
```

> If you logged out, repeat the level 25 → 26 trick to get a shell as bandit26 again: bandit26's login shell is still `showtext`, you can't log in straight to a bash.
{: .prompt-info }

## The challenge

The [level goal](https://overthewire.org/wargames/bandit/bandit27.html) says there's a **setuid binary** in the home that lets us run commands as bandit27. It's exactly the same mechanism as level 19 → 20.

I check the permissions to confirm:

```console
bandit26@bandit:~$ ls -l bandit27-do
-rwsr-x--- 1 bandit27 bandit26 ... bandit27-do
```

There's the **SUID bit** `s` in the owner's permissions, and the owner is **bandit27**. That means running the binary makes the process run with bandit27's identity, not ours.

## The solution

First I run it without arguments to see how it's used:

```console
bandit26@bandit:~$ ./bandit27-do
Run a command as another user.
  Example: ./bandit27-do id
```

It's the same launcher as always: you pass it a command and it runs it as bandit27. So I ask it to read bandit27's password, which bandit27 can read:

```console
bandit26@bandit:~$ ./bandit27-do cat /etc/bandit_pass/bandit27
[REDACTED]
```

And there's bandit27's password.

> Why does it work? The SUID bit makes the program run with its **owner's** permissions (bandit27), not those of whoever launches it (bandit26). Since the `cat` ends up being run by bandit27, it can read `/etc/bandit_pass/bandit27`, even though we as bandit26 couldn't. I explain it in depth in the [19 → 20 walkthrough](/en/posts/bandit-level-19-20/).
{: .prompt-tip }

## Why this matters in security

This level chains two concepts you've seen separately: first **escaping a restricted shell** (level 25 → 26) and then **abusing a SUID binary** to move to another user. In a real pentest you rarely have a single clean path: the norm is chaining small flaws (a poorly restricted shell + a permissive SUID) until you get where you want. That's why, as soon as you get a new shell, the first thing is usually to enumerate SUID binaries:

```console
bandit26@bandit:~$ find / -perm -4000 -type f 2>/dev/null
```

## Moving to the next level

```console
bandit26@bandit:~$ ssh bandit27@bandit.labs.overthewire.org -p 2220
```

Log in with the bandit27 password you just recovered.

> The next level changes topic and enters new territory: **Git**. From here on, several levels are about finding secrets in a repository's history.
{: .prompt-info }

## Summary

| Command / concept | What it's for |
| --- | --- |
| SUID bit (`-rws`) | Runs the program with its owner's permissions |
| `ls -l file` | See permissions and spot the SUID `s` |
| `./bandit27-do command` | Run a command as bandit27 |
| `find / -perm -4000 -type f 2>/dev/null` | Enumerate SUID binaries on the system |
| Chaining flaws | Broken restricted shell + SUID = escalation between users |

{% include series-nav.html %}
