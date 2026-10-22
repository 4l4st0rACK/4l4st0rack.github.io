---
title: "OverTheWire Bandit: Level 27 → 28 Walkthrough"
date: 2026-10-22 17:28:00 +0200
categories: [Linux, Bandit]
tags: [bandit, overthewire, linux, git, ssh, awk]
description: OverTheWire Bandit level 27 to 28 walkthrough, cloning a Git repository over SSH from your local machine, with why you can't clone from the server itself.
ref: bandit-27-28
serie: bandit
nivel: 28
---

The **Git** block begins. From here on, several levels are about the same thing: clone a repository and look for secrets in it. This first one is the simplest (the password is in plain sight in the README), but it has a connection gotcha that trips up a lot of people, so let's understand it properly.

> Level passwords are shown as **[REDACTED]**. Following OverTheWire's rules, I don't publish credentials: the fun part is getting there yourself.
{: .prompt-warning }

## The challenge

The [level goal](https://overthewire.org/wargames/bandit/bandit28.html) says there's a **Git** repository at `ssh://bandit27-git@localhost/home/bandit27-git/repo`, and you have to clone it to find the next level's password. The credential for `bandit27-git` is bandit27's password.

## The gotcha: where you clone from

The first thing everyone intuitively does is SSH into Bandit as bandit27 and clone there. **It doesn't work**, and the error explains itself:

```text
!!! You are trying to log into this SSH server from localhost.
!!! Connecting from/to localhost is blocked to conserve resources.
!!! Please log out and log in again, directly from your client machine.
```

Why? When you're **inside** the Bandit server, the host `bandit.labs.overthewire.org` (or `localhost`) resolves to `127.0.0.1`, i.e. the machine itself. OverTheWire **blocks localhost-to-itself connections** to avoid wasting resources. Cloning the repo opens a new SSH connection (git uses SSH under the hood), and that connection would be localhost → localhost: blocked.

The fix is to clone **from your own machine**, not from inside the Bandit session. From your machine, `bandit.labs.overthewire.org` is a normal remote host and the connection is allowed.

> The prompt is your best clue: if it says `bandit27@bandit:~$`, you're **inside** the server and you'll hit the block. If it shows your own username and machine, you're where you need to be. Run `exit` until you're back at your local terminal.
{: .prompt-info }

## The solution

On **my machine**, I create a temporary directory and clone with the full URL, including the **port 2220**:

```console
$ cd $(mktemp -d)
$ git clone ssh://bandit27-git@bandit.labs.overthewire.org:2220/home/bandit27-git/repo
Cloning into 'repo'...
```

It asks for `bandit27-git`'s password (bandit27's), I enter it and it clones. Inside there's a `README`:

```console
$ cd repo
$ ls
README
$ cat README
The password to the next level is [REDACTED]
```

The password is in plain text. As always, to get **only** the password without the surrounding text:

```console
$ cat README | awk '{print $NF}'
[REDACTED]
```

`awk '{print $NF}'` prints the line's last field (`$NF` = *number of fields*), i.e. the last word, which is exactly the password.

## Important detail: the port in a Git URL

A classic stumble: in an `ssh://` URL the port goes **attached to the host with a colon** (`host:2220`), **not** with `-p` like the `ssh` command. If you omit the port, git uses 22 by default and the Bandit server rejects you (it only listens for the game on 2220).

There's a "short" scp-like form (`user@host:path`), but that one **doesn't accept a port**. That's why here we have to use the long form `ssh://...:2220/...`.

## Other ways to do it

### 1. Pass the port to git via an environment variable

If you prefer the short syntax, you can tell SSH the port with `GIT_SSH_COMMAND`:

```console
$ GIT_SSH_COMMAND='ssh -p 2220' git clone bandit27-git@bandit.labs.overthewire.org:/home/bandit27-git/repo
```

Here you pass `-p 2220` to the `ssh` git uses under the hood, so you can use the short form.

### 2. Set it permanently in your `~/.ssh/config`

If you're going to work on Bandit a lot, define the host once:

```
Host bandit-git
    HostName bandit.labs.overthewire.org
    Port 2220
    User bandit27-git
```

And clone without repeating port or user:

```console
$ git clone ssh://bandit-git/home/bandit27-git/repo
```

## Why this matters in security

This level is trivial on purpose: the secret is in the README. But the underlying message is huge: **Git repositories are a gold mine of credentials**. Passwords, API tokens, private keys... get committed by mistake all the time. And here's the interesting part we'll exploit in the next levels: even if you delete a secret in a later commit, it **stays alive in the history** (`git log`, `git diff`, branches, stash...). Cloning a repo gives you all that history, not just the current version.

> That's why, in a pentest, when you find a repository (or an exposed `.git` on a website), the first thing is to review the full history, not just the current files. Tools like `gitleaks` or `trufflehog` automate exactly that search for secrets in the history.
{: .prompt-tip }

## Moving to the next level

```console
$ sshpass -p '[REDACTED]' ssh bandit28@bandit.labs.overthewire.org -p 2220
```

> The next level stays with Git but raises the bar: the current password is "censored" in the README, and you'll have to dig into the **commit history** to find the version where it was still visible.
{: .prompt-info }

## Summary

| Command / concept | What it's for |
| --- | --- |
| `git clone ssh://user@host:2220/path` | Clone a repo over SSH specifying the port |
| localhost block | OverTheWire rejects localhost → localhost connections |
| `cd $(mktemp -d)` | Work in a writable temporary directory |
| `awk '{print $NF}'` | Print the last word of a line |
| `GIT_SSH_COMMAND='ssh -p 2220'` | Pass the port to git with the short syntax |
| `~/.ssh/config` | Save host, port and user so you don't repeat them |
| Git history | Keeps old versions: deleted secrets are still there |

{% include series-nav.html %}
