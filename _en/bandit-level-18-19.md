---
title: "OverTheWire Bandit: Level 18 → 19 Walkthrough"
date: 2026-10-19 20:35:00 +0200
categories: [Linux, Bandit]
tags: [bandit, overthewire, linux, ssh, bashrc, scp]
description: OverTheWire Bandit level 18 to 19 walkthrough, getting past a modified .bashrc that logs you out over SSH by running remote commands without an interactive shell.
ref: bandit-18-19
serie: bandit
nivel: 19
---

This level has a twist: we have the right password, but as soon as we log in over SSH **we get kicked out**. Someone tampered with the user's `.bashrc` to end the session right away. The key is understanding **when `.bashrc` runs** and how to use SSH to run commands without opening an interactive session.

> Level passwords are shown as **[REDACTED]**. Following OverTheWire's rules, I don't publish credentials: the fun part is getting there yourself.
{: .prompt-warning }

## The challenge

The [level goal](https://overthewire.org/wargames/bandit/bandit19.html) says the password for the next level is in a `readme` file in the home directory, but someone modified `.bashrc` to **log you out when you log in with SSH**.

We already warned about this at the end of the [previous level](/en/posts/bandit-level-17-18/).

## First attempt: connecting as usual

```console
$ sshpass -p '[REDACTED]' ssh bandit18@bandit.labs.overthewire.org -p 2220
```

Authentication works, the OverTheWire welcome banner shows up... and right after that the connection closes. We never see the `bandit18@bandit:~$` prompt, so there isn't even time for the usual `export TERM=xterm`.

Why? When you log in over SSH the normal way, the server opens an **interactive bash shell**, and when bash starts an interactive shell it runs `~/.bashrc`. If someone put an `exit` in that file, the shell closes as soon as it opens, and your connection goes with it.

## The solution: giving SSH a command

`ssh` isn't just for opening a session: if you write a command after the destination, **it runs it on the server and disconnects**, without opening an interactive shell. So I ask it to run `bash`:

```console
$ sshpass -p '[REDACTED]' ssh bandit18@bandit.labs.overthewire.org -p 2220 bash
```

This time it doesn't kick me out. The screen does go "blank", though: **there's no prompt**. But the shell is there listening, and if I type commands, it runs them:

```console
ls
readme
cat readme
[REDACTED]
```

And there's bandit19's password.

Why does it work? When SSH runs a command, it doesn't allocate you a terminal (TTY), and the `bash` we launch starts as a **non-interactive shell**: it reads whatever you type on standard input and runs it, with no prompt and no colors. And non-interactive shells **don't run `~/.bashrc`** (on top of that, Ubuntu's `.bashrc` starts with a check that returns immediately if the shell isn't interactive). Result: the booby-trapped `exit` never runs.

> In this "promptless shell", `export TERM=xterm` makes no sense: there's no terminal to configure. Full-screen programs like `nano` or `less` won't work well either. To read a file, `cat` is more than enough.
{: .prompt-info }

## Other ways to do it

### 1. Run exactly the command you need

If you already know what you want to do, you don't need to open any shell: give SSH the command and it returns the output.

```console
$ sshpass -p '[REDACTED]' ssh bandit18@bandit.labs.overthewire.org -p 2220 cat readme
[REDACTED]
```

You can even chain several inside quotes so they all run on the server:

```console
$ sshpass -p '[REDACTED]' ssh bandit18@bandit.labs.overthewire.org -p 2220 'ls -la; cat readme'
```

With `ls -la` you'll also see `.bashrc`. If you `cat .bashrc` the same way, you can find the line that kicks you out at the end of the file. It's a way to see the trap with your own eyes.

### 2. A real interactive shell, without `.bashrc`

If you want a normal prompt, you can force SSH to allocate a terminal with `-t` and ask for a shell that doesn't load `.bashrc`:

```console
$ sshpass -p '[REDACTED]' ssh -t bandit18@bandit.labs.overthewire.org -p 2220 /bin/sh
$ cat readme
[REDACTED]
```

`/bin/sh` is a different shell from bash and **doesn't read `~/.bashrc`**, so the trap never fires. With `-t` you also get a TTY, so you see a prompt (a simpler one, just `$`). Same idea with bash:

```console
$ sshpass -p '[REDACTED]' ssh -t bandit18@bandit.labs.overthewire.org -p 2220 bash --norc
```

`--norc` literally tells bash not to load `.bashrc`.

### 3. Grab the file with `scp`

`scp` copies files over SSH, and since it doesn't open an interactive shell, `.bashrc` doesn't affect it. You can copy `readme` to your machine and read it there:

```console
$ sshpass -p '[REDACTED]' scp -P 2220 bandit18@bandit.labs.overthewire.org:readme .
$ cat readme
[REDACTED]
```

> Watch out: `scp` uses an **uppercase** `-P` for the port, the opposite of `ssh`, which uses a lowercase `-p`. A very common mistake.
{: .prompt-tip }

## Moving to the next level

Once out of bandit18 (with the promptless shell method, `exit` or `Ctrl+D` is enough), I log in to the next level:

```console
$ sshpass -p '[REDACTED]' ssh bandit19@bandit.labs.overthewire.org -p 2220
```

> The next level introduces a very important concept for privilege escalation: **setuid binaries**, programs that run with their owner's permissions instead of yours.
{: .prompt-info }

## Summary

| Command / concept | What it's for |
| --- | --- |
| `~/.bashrc` | Script bash runs when it opens an interactive shell |
| `ssh user@host -p 2220 command` | Run a command on the server without an interactive session |
| `ssh ... bash` | Non-interactive shell: no prompt, but it runs what you type |
| `ssh -t` | Force a terminal (TTY) to be allocated |
| `/bin/sh` / `bash --norc` | Shells that don't load `.bashrc` |
| `scp -P 2220 user@host:file .` | Copy a file from the server to your machine |

{% include series-nav.html %}
