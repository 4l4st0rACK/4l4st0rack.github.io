---
title: "OverTheWire Bandit: Level 25 → 26 Walkthrough"
date: 2026-10-20 21:05:00 +0200
categories: [Linux, Bandit]
tags: [bandit, overthewire, linux, shell, restricted-shell, more, vim, ssh]
description: OverTheWire Bandit level 25 to 26 walkthrough, escaping a restricted shell based on the more pager to get a real shell as bandit26.
ref: bandit-25-26
serie: bandit
nivel: 26
---

This is one of the most fun levels in Bandit. We have the key to log in as bandit26, but its shell **isn't a normal shell**: it's a pager in disguise. The challenge is to **escape that restricted shell** and get a real one. It's a very real technique: restricted shells are used to lock users in, and knowing how to break out of them is part of the job.

> Level passwords and keys are shown as **[REDACTED]**. Following OverTheWire's rules, I don't publish credentials: the fun part is getting there yourself.
{: .prompt-warning }

## Connecting

```console
$ sshpass -p '[REDACTED]' ssh bandit25@bandit.labs.overthewire.org -p 2220
bandit25@bandit:~$ export TERM=xterm
bandit25@bandit:~$ ls
bandit26.sshkey
```

## The challenge

The [level goal](https://overthewire.org/wargames/bandit/bandit26.html) says logging in as bandit26 **won't be as easy as before**, but the key to get in is in bandit25's home. Sure enough, there's `bandit26.sshkey`: an SSH private key, like the ones in earlier levels.

## First attempt: log in with the key

I bring the key to my machine with `scp` (remember: uppercase `-P` for the port):

```console
$ scp -P 2220 bandit25@bandit.labs.overthewire.org:/home/bandit25/bandit26.sshkey .
$ chmod 600 bandit26.sshkey
$ ssh -i bandit26.sshkey bandit26@bandit.labs.overthewire.org -p 2220
```

Authentication works... but the session closes immediately. We never get a prompt. Why?

## Why it kicks us out: bandit26's shell

A user's login shell is defined in `/etc/passwd`. I go back to the bandit25 session and look at bandit26's:

```console
bandit25@bandit:~$ cat /etc/passwd | grep bandit26
bandit26:x:11026:11026:bandit26,,,:/home/bandit26:/usr/bin/showtext
```

The last field is the shell, and instead of `/bin/bash` it says **`/usr/bin/showtext`**. Let's see what that is:

```console
bandit25@bandit:~$ cat /usr/bin/showtext
#!/bin/sh

export TERM=linux

exec more ~/text.txt
exit 0
```

It's a script that runs **`more ~/text.txt`** and nothing else. `more` is a pager (it shows files page by page). When it finishes, the `exec` makes the session close. That's a **restricted shell**: bandit26 isn't given a command interpreter, just shown a file and then kicked out.

## The solution: escape from `more`

The key is how `more` behaves:

- If the file **fits entirely on screen**, `more` shows it and exits (it doesn't go interactive). That's why the session closes so fast: `text.txt` is very short.
- If the file **doesn't fit**, `more` goes **interactive** (the classic `--More--` at the bottom) waiting for commands.

So the trick is to force `more` into interactive mode. How? **By making the terminal window very small** before connecting, so `text.txt` doesn't fit:

1. Shrink your terminal window to very few rows tall (3-4 lines).
2. Connect: `ssh -i bandit26.sshkey bandit26@bandit.labs.overthewire.org -p 2220`.
3. Now `more` can't show everything and goes interactive.

From `more`'s interactive mode you can open an editor: pressing **`v`** opens `vim` on the file. And once inside `vim` we have room to enlarge the terminal again and, crucially, **vim lets us run a shell**:

```vim
:set shell=/bin/bash
:shell
```

- `:set shell=/bin/bash` tells vim which shell to use.
- `:shell` launches it.

And we're in as **bandit26** with a real bash:

```console
bandit26@bandit:~$ whoami
bandit26
```

> The pattern "a program that shouldn't give you a shell but can launch another program that does" is exactly what [GTFOBins](https://gtfobins.github.io/) catalogs. `more`, `less`, `vim`, `man`, `find`... they all have ways to escape to a shell. It's one of the first things you check when you hit a restricted shell.
{: .prompt-tip }

## Grab bandit26's password

Important: we logged in to level 25 with bandit26's **private key**, not its password, so we still don't have it. Now that we're bandit26, we read it and save it so we don't depend on the key:

```console
bandit26@bandit:~$ cat /etc/bandit_pass/bandit26
[REDACTED]
```

## Moving to the next level

We now have a shell as bandit26, which is exactly what we need for the next level (there's a SUID binary in the home). No need to even log out.

> The next level builds on this very shell: a SUID binary, `bandit27-do`, that runs commands as bandit27. We cover it in the 26 → 27 walkthrough.
{: .prompt-info }

## Summary

| Command / concept | What it's for |
| --- | --- |
| `/etc/passwd` | Defines, in the last field, each user's login shell |
| Restricted shell | A shell that limits what the user can do (here, only `more`) |
| `more` (small screen) | If the file doesn't fit, it goes interactive |
| `v` in `more` | Open the file in `vim` |
| `:set shell=/bin/bash` + `:shell` in vim | Launch a shell from the editor |
| [GTFOBins](https://gtfobins.github.io/) | Catalog of binaries to escape restricted shells with |

{% include series-nav.html %}
