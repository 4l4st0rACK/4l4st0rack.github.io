---
title: "OverTheWire Bandit: Level 13 → 14 Walkthrough"
date: 2026-10-18 22:25:00 +0200
categories: [Linux, Bandit]
tags: [bandit, overthewire, linux, ssh, ssh-keys, chmod, permissions]
description: OverTheWire Bandit level 13 to 14 walkthrough. No password this time, but an SSH private key. Learn how key authentication works, why the key needs 600 permissions and how to log in with ssh -i.
ref: bandit-13-14
serie: bandit
nivel: 14
---

Until now, every level gave us a **password** to log into the next one. This time the game changes: what we find is an **SSH private key**, which lets us log in as `bandit14` without typing any password. This is the authentication method used on real servers, so it's worth understanding properly.

> Passwords are shown as `[REDACTED]`, following the [OverTheWire rules](https://overthewire.org/rules/). For the same reason, we don't publish the contents of the private key either: with it, anyone could log in as `bandit14`.
{: .prompt-warning }

## Connecting to level 13

```console
$ sshpass -p '[REDACTED]' ssh bandit13@bandit.labs.overthewire.org -p 2220
bandit13@bandit:~$ export TERM=xterm
```

## The challenge

According to the [level 14 page](https://overthewire.org/wargames/bandit/bandit14.html), the password for the next level is stored in `/etc/bandit_pass/bandit14` and **can only be read by user `bandit14`**. For this level we don't get `bandit14`'s password, but an SSH private key to log in as that user.

```console
bandit13@bandit:~$ ls
HINT  sshkey.private
```

This time there are two files. Let's start with the hint:

```console
bandit13@bandit:~$ cat HINT
If you have trouble with this level, note the following:

1) As for all other levels, this level has a website with information:
https://overthewire.org/wargames/bandit/bandit14.html
2) No, the level is not broken. To verify, see:
https://status.overthewire.org/
3) The current version of OverTheWire prevents logging in from one
level to another via localhost. Log out, and see 1)
4) If you get errors, read the error message on your screen.
We mean it!
```

The key point is number **3**: we can't use the key from the server itself to jump to `bandit14` through `localhost` (that's how this level was solved in older versions of Bandit). We have to **log out** and connect from our own machine using the key.

## SSH keys: how they work

Key-based authentication relies on a **key pair** that always goes together:

| Key | Where it lives | What it does |
|---|---|---|
| **Public** | On the server, in the user's `~/.ssh/authorized_keys` | Works like a lock: it can be visible to everyone |
| **Private** | In your possession (here, `sshkey.private`) | It's the key: whoever has it can get in, so it's never shared |

When you connect, the server gives you a challenge that can only be solved with the private key matching one of its authorized public keys. The private key **never travels over the network**: it's only used to prove you have it. If the challenge succeeds, you're in without a password.

In Bandit, someone has left `bandit14`'s private key in `bandit13`'s home directory, and that's exactly what we're going to use.

## Bringing the key to our machine

We print the key with `cat`:

```console
bandit13@bandit:~$ cat sshkey.private
[REDACTED: private key]
```

We copy the whole content, including the `-----BEGIN ...-----` and `-----END ...-----` lines, log out of the server and paste it into a file called `sshkey.private` on our machine.

## 600 permissions: why they're required

```console
$ chmod 600 sshkey.private
```

This step isn't optional. SSH checks the private key's permissions before using it and, if other users on the system can read it, **refuses to use it**: it shows an *unprotected private key file* warning saying the permissions are too open and ignores the key. It makes sense: a private key anyone can read is no longer private.

`600` is the **octal** notation for permissions. Each digit corresponds to one of the three levels we saw in [level 6 → 7](/en/posts/bandit-level-6-7/): owner, group and others. And each digit is the sum of the permissions it grants:

| Permission | Value |
|---|---|
| Read (`r`) | 4 |
| Write (`w`) | 2 |
| Execute (`x`) | 1 |

| Digit | For whom | Sum | Permissions |
|---|---|---|---|
| `6` | Owner | 4 + 2 | read and write (`rw-`) |
| `0` | Group | 0 | none (`---`) |
| `0` | Others | 0 | none (`---`) |

In other words, `600` means "only I can read and modify this file". In an `ls -l` listing it shows up as `-rw-------`.

## Logging in as bandit14 with the key

```console
$ ssh -i sshkey.private bandit14@bandit.labs.overthewire.org -p 2220
bandit14@bandit:~$
```

The `-i` (*identity file*) option tells `ssh` which private key to use. Notice we don't need `sshpass` here: there's no password to pass, the key does all the work and we get straight in.

## Reading bandit14's password

Now that we're `bandit14`, we can read the file the challenge pointed us to:

```console
bandit14@bandit:~$ cat /etc/bandit_pass/bandit14
[REDACTED]
```

`/etc/bandit_pass/` holds every level's password, and each file can only be read by its own user. From `bandit13` we couldn't have opened this one; as `bandit14`, we can. Save it: it's what we'll use to connect to this level from now on, and we'll need it in the next one.

## Other ways to do it

### 1. Fetch the key with scp

```console
$ sshpass -p '[REDACTED]' scp -P 2220 bandit13@bandit.labs.overthewire.org:sshkey.private .
$ chmod 600 sshkey.private
```

Just like with the hexdump in the [previous level](/en/posts/bandit-level-12-13/), `scp` avoids copy-and-paste mistakes. With keys this matters even more: if a character or the trailing newline gets lost when pasting, `ssh` will say the key has an invalid format and you won't get in. Remember: uppercase `-P` for the port in `scp`.

### 2. Get the password without opening an interactive session

```console
$ ssh -i sshkey.private bandit14@bandit.labs.overthewire.org -p 2220 cat /etc/bandit_pass/bandit14
[REDACTED]
```

If you add a command after the address, `ssh` doesn't open a terminal: it runs that command on the server, shows you its output and closes the connection. Very handy for automating tasks on remote machines.

### 3. Check which public key it matches

```console
$ ssh-keygen -y -f sshkey.private
```

`ssh-keygen -y` derives the **public key** from the private one (the reverse is impossible, and that's what the whole system's security relies on). It's a quick way to check the key was copied correctly: if the file is corrupt, this command fails before you even try `ssh`. What it prints is what would be in the server's `authorized_keys`.

## Moving on to the next level

We're already logged in as `bandit14`, and with its password we can connect as usual in future sessions:

```console
$ sshpass -p '[REDACTED]' ssh bandit14@bandit.labs.overthewire.org -p 2220
```

## Summary

| Command / concept | What it's for |
|---|---|
| Public / private key | A key pair: the public one on the server, the private one only in your hands |
| `ssh -i key user@host` | Log in with a private key instead of a password |
| `chmod 600 file` | Only the owner can read and write (required for private keys) |
| Octal permissions | r = 4, w = 2, x = 1; one digit each for owner, group and others |
| `ssh ... command` | Run a remote command without an interactive session |
| `scp -P port source destination` | Copy files over SSH |
| `ssh-keygen -y -f key` | Derive the public key from the private key |
| `/etc/bandit_pass/banditN` | Each level's password, readable only by its user |

{% include series-nav.html %}
