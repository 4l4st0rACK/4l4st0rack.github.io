---
title: "OverTheWire Bandit: Level 0 → 1 Walkthrough"
date: 2026-10-18 12:00:00 +0200
categories: [Linux, Bandit]
tags: [bandit, overthewire, ssh, sshpass, awk, linux]
description: Starting the OverTheWire Bandit wargame. We log in over SSH, fix the terminal and extract the level 1 password by filtering a file with tail and awk.
ref: bandit-0-1
serie: bandit
nivel: 1
---

**Bandit** is the [OverTheWire](https://overthewire.org/wargames/bandit/) wargame designed for absolute beginners. Each level is a small challenge on a Linux server. To beat it you have to find the password for the next level, and finding it teaches you how to use the terminal.

In this series I solve the levels one by one. I don't stop at the command you need to type: I explain **why** it works and look for the cleanest way to do it.

> Following the [OverTheWire rules](https://overthewire.org/rules/), level passwords are not published. In the examples they show up as `[REDACTED]`. The fun part is getting them yourself.
{: .prompt-warning }

## What you need

- **A terminal with SSH.** Linux and macOS ship with one. On Windows 10/11, PowerShell includes `ssh` out of the box, although for this series I recommend **WSL**.
- **`sshpass`** (optional). It lets you hand the password to SSH in the same command instead of typing it at the prompt. Since you'll be connecting a lot, it saves time:

```console
$ sudo apt install sshpass
```

## Connecting to level 0

The connection details are on the [level 0 page](https://overthewire.org/wargames/bandit/bandit0.html): host `bandit.labs.overthewire.org`, port `2220`, user `bandit0` and password `bandit0`. It's the only password the website gives you; from here on, you earn each one by solving the previous level.

```console
$ sshpass -p 'bandit0' ssh bandit0@bandit.labs.overthewire.org -p 2220
```

Breaking it down:

- `sshpass -p 'bandit0'`: sshpass feeds the password `bandit0` to SSH when it asks for it.
- `ssh bandit0@bandit.labs.overthewire.org`: connect as the **user** `bandit0` to the **host** `bandit.labs.overthewire.org`, using the `user@host` format.
- `-p 2220`: the **port**. SSH uses 22 by default, but Bandit listens on 2220.

Notice that **`-p` shows up twice and means different things**. The first one belongs to `sshpass` and is the password. The second one belongs to `ssh` and is the port. Each option belongs to the command right before it.

> **If sshpass does nothing the first time:** the first time you connect to a server, SSH asks whether you trust its fingerprint and waits for a `yes`. sshpass can't answer that question, so it hangs or exits without connecting. Connect once with plain `ssh` and accept with `yes`, or add `-o StrictHostKeyChecking=accept-new` to the command to accept new hosts automatically.
{: .prompt-tip }

> Passing passwords on the command line is handy in a lab, but **in a real environment it's bad practice**: the password ends up in your shell history and other users on the machine can see it in the process list. Real systems use SSH keys.
{: .prompt-danger }

## Fixing the terminal: `export TERM=xterm`

Right after logging in, the first thing I do on every level is this:

```console
bandit0@bandit:~$ export TERM=xterm
```

The `TERM` environment variable tells programs what kind of terminal you're using, so they know how to draw on screen. When you connect over SSH, the server inherits the value from your local terminal. If the server doesn't recognise that terminal type, commands like `clear` or editors like `nano` break or render badly. With `export TERM=xterm` we tell it to use `xterm`, a type every system knows.

`export` makes the variable available to any program you launch from that shell. It only lasts for the session, which is why you have to repeat it every time you log into a level.

## Finding the level 1 password

The [level 1 page](https://overthewire.org/wargames/bandit/bandit1.html) says the password is stored in a file called **`readme`** in the home directory. Let's see what's there:

```console
bandit0@bandit:~$ ls
readme
```

`cat` shows its contents:

```console
bandit0@bandit:~$ cat readme
```

You get a welcome message from OverTheWire and, on the last line with text, the password. We could copy it by hand and move on, but the point of this series is learning the terminal. So let's extract **only the password**.

## Filtering the output: `cat | tail | awk`

```console
bandit0@bandit:~$ cat readme | tail -n 2 | awk '{print $NF}'
[REDACTED]
```

The vertical bar `|` is a **pipe**: it takes the output of one command and passes it as input to the next. That's how we chain small tools, each doing a single job:

1. **`cat readme`** prints the whole file.
2. **`tail -n 2`** keeps only the **last 2 lines**. We take two instead of one because the file ends with a blank line, and the password is on the second-to-last one.
3. **`awk '{print $NF}'`** prints the **last field** of each line. `awk` splits each line on whitespace. `NF` (*Number of Fields*) is how many pieces the line has, so `$NF` is the last one. On the password line, the last piece is the password itself. On the blank line there's nothing to print.

> **Shorter version:** `awk` can read the file directly and filter by content, with no `cat` or `tail`: `awk '/password/ {print $NF}' readme`. It prints the last field only on lines containing the word *password*. Chaining commands is the best way to learn, but it's worth knowing there's often a shortcut.
{: .prompt-info }

## Moving on to level 1

Password in hand, we log out and connect as `bandit1`:

```console
bandit0@bandit:~$ exit
$ sshpass -p '[REDACTED]' ssh bandit1@bandit.labs.overthewire.org -p 2220
```

> Keep the passwords in a notes file on your machine. Bandit won't remember them for you.
{: .prompt-tip }

## Summary

| Command | What it does |
|---|---|
| `sshpass -p 'pass' ssh user@host -p port` | Connect over SSH passing the password in the command |
| `export TERM=xterm` | Tell the server which terminal type we use, so `clear` and friends work |
| `ls` | List files |
| `cat file` | Print a file's contents |
| `command1 \| command2` | Feed one command's output into another |
| `tail -n N` | Keep the last N lines |
| `awk '{print $NF}'` | Print the last field of each line |

{% include series-nav.html %}
