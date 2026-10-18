---
title: "OverTheWire Bandit: Level 6 → 7 Walkthrough"
date: 2026-10-18 20:00:00 +0200
categories: [Linux, Bandit]
tags: [bandit, overthewire, linux, find, redirection, stderr]
description: OverTheWire Bandit level 6 to 7 walkthrough. Search the whole server with find by owner, group and size, and silence permission errors by redirecting stderr to /dev/null.
ref: bandit-6-7
serie: bandit
nivel: 7
---

In the [previous level](/en/posts/bandit-level-5-6/) we used `find` inside a specific folder. Now the challenge raises the stakes: the file is **somewhere on the server**, with no hint about the directory. All we get is its "description": owner, group and size. Time to search from the root of the filesystem and, along the way, learn how to get rid of the hundreds of permission errors that come with it.

> Passwords are shown as `[REDACTED]`, following the [OverTheWire rules](https://overthewire.org/rules/).
{: .prompt-warning }

## Connecting to level 6

```console
$ sshpass -p '[REDACTED]' ssh bandit6@bandit.labs.overthewire.org -p 2220
bandit6@bandit:~$ export TERM=xterm
```

As always, `sshpass` passes the password to SSH so we don't have to type it, and `export TERM=xterm` avoids unknown-terminal warnings when using commands like `clear`.

## The challenge

According to the [level 7 page](https://overthewire.org/wargames/bandit/bandit7.html), the password is stored somewhere on the server in a file with these properties:

- owned by user **bandit7**
- owned by group **bandit6**
- **33 bytes** in size

There's no point running `ls` in our home directory here: the file could be anywhere. We need to walk the entire filesystem.

## Searching the whole server with find

```console
bandit6@bandit:~$ find / -type f -user bandit7 -group bandit6 -size 33c 2>/dev/null
/var/lib/dpkg/info/bandit7.password
```

A single result. Let's break the command down piece by piece:

| Part | What it does |
|---|---|
| `/` | Starts searching at the root, i.e. the **entire** filesystem. |
| `-type f` | Regular files only (skips directories, links, etc.). |
| `-user bandit7` | The file's **owner** must be `bandit7`. |
| `-group bandit6` | The file's **group** must be `bandit6`. |
| `-size 33c` | Exactly 33 **bytes** in size. |
| `2>/dev/null` | Sends error messages to the system's "trash can". |

`find` joins all the tests with an implicit logical **AND**: the file has to match all four at once.

### Watch out for the `c` in `-size`

We already used it in the previous level with `-size 1033c`, but it's worth repeating: **without a suffix, `find` counts in 512-byte blocks**, not bytes. So `-size 33` would look for files of roughly 33 × 512 = 16,896 bytes and find nothing. The `c` (for *char*, one byte per character) is what makes the search use exact bytes.

And 33 bytes makes sense: Bandit passwords are 32 characters long, plus the trailing newline.

### Why `2>/dev/null`?

In Linux every process has three standard streams, each identified by a number (*file descriptor*):

| Descriptor | Name | Used for |
|---|---|---|
| `0` | stdin | Input |
| `1` | stdout | Normal output (the results) |
| `2` | stderr | Error messages |

When we search from `/` as `bandit6`, `find` tries to enter lots of directories we have no permission for: `/root`, other users' home directories, parts of `/proc`... Each one produces a `Permission denied` line on **stderr**. Without a redirect, the one good result gets buried under hundreds of errors.

With `2>` we tell the shell "whatever comes out of stream 2, send it here", and `/dev/null` is a special file that swallows everything you write to it. The result: the errors vanish and only **stdout** stays on screen, which is exactly what we want.

> `2>/dev/null` doesn't give `find` more permissions or make it search better: the errors still happen, we just don't see them. That's why it's safe to use here, where we know those errors tell us nothing useful.
{: .prompt-info }

## Reading the password

```console
bandit6@bandit:~$ cat /var/lib/dpkg/info/bandit7.password
[REDACTED]
```

Two interesting details:

- **The hiding place.** `/var/lib/dpkg/info/` is where `dpkg`, the Debian/Ubuntu package manager, keeps information about installed packages (file lists, checksums...). There are hundreds of files in there, so it's a great spot to blend one more in. Looking for it by name would have been a nightmare; searching by properties finds it instantly.
- **Why can we read a file that isn't ours?** The owner is `bandit7`, but the group is `bandit6`, which is our user. Linux permissions are checked at three levels (owner, group and others), and what lets us read it is the **group** read permission. That's why the challenge gives us the group as a hint: it's the "door" that was left open for us.

## Other ways to do it

### 1. Find and read in one step with `-exec`

```console
bandit6@bandit:~$ find / -type f -user bandit7 -group bandit6 -size 33c -exec cat {} \; 2>/dev/null
[REDACTED]
```

`-exec` runs a command on each file found. `{}` is replaced with the file's path and `\;` marks the end of the command (the backslash escapes the `;` so the shell doesn't eat it). With many results, `-exec cat {} +` would pass all the paths to a single `cat` at once, which is more efficient.

### 2. Check the permissions with `-ls`

```console
bandit6@bandit:~$ find / -type f -user bandit7 -group bandit6 -size 33c -ls 2>/dev/null
```

`-ls` prints each result in a format similar to `ls -l`: permissions, owner, group, size and date. It's a good way to **verify** that the file matches the conditions and to see for yourself the group read permission mentioned above. You'd get the same with `ls -l /var/lib/dpkg/info/bandit7.password`.

### 3. Filter the errors with `grep` instead of discarding them

```console
bandit6@bandit:~$ find / -type f -user bandit7 -group bandit6 -size 33c 2>&1 | grep -v "Permission denied"
```

Here we do the opposite: `2>&1` **merges** stderr into stdout (sends stream 2 wherever stream 1 goes), so everything goes through the pipe, and `grep -v` drops the lines containing "Permission denied". It's longer than `2>/dev/null`, but it has one advantage: if a **different** (and maybe interesting) error showed up, you'd still see it.

## Moving on to the next level

```console
bandit6@bandit:~$ exit
$ sshpass -p '[REDACTED]' ssh bandit7@bandit.labs.overthewire.org -p 2220
```

## Summary

| Command / concept | What it's for |
|---|---|
| `find /` | Search the entire filesystem |
| `-user` / `-group` | Filter by owner and by group |
| `-size 33c` | Exact size in bytes (without `c`, it counts 512-byte blocks) |
| `2>/dev/null` | Discard error messages (stderr) |
| `2>&1` | Merge stderr into stdout to process them together |
| `-exec cmd {} \;` | Run a command on each result |
| `-ls` | Show permissions, owner, group and size of each result |
| Group permissions | Let us read files owned by someone else if we share the group |

{% include series-nav.html %}
