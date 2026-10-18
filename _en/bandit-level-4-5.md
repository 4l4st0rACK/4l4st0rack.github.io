---
title: "OverTheWire Bandit: Level 4 → 5 Walkthrough"
date: 2026-10-18 16:00:00 +0200
categories: [Linux, Bandit]
tags: [bandit, overthewire, linux, find, file, xargs]
description: Ten files and only one contains human-readable text. Using find, file and xargs to spot it without opening them one by one.
ref: bandit-4-5
serie: bandit
nivel: 5
---

This is where it gets interesting: it's no longer enough to open a file, you have to **find** the right one among several. We could open them one by one, but the point is to let the terminal do it for us. For that we combine three tools you'll use constantly: `find`, `file` and `xargs`.

> Passwords show up as `[REDACTED]`, following the [OverTheWire rules](https://overthewire.org/rules/).
{: .prompt-warning }

## Connecting to level 4

```console
$ sshpass -p '[REDACTED]' ssh bandit4@bandit.labs.overthewire.org -p 2220
bandit4@bandit:~$ export TERM=xterm
```

## The goal

The [level 5 page](https://overthewire.org/wargames/bandit/bandit5.html) says the password is in **the only human-readable file** in the `inhere` directory:

```console
bandit4@bandit:~$ ls
inhere
bandit4@bandit:~$ cd inhere
bandit4@bandit:~/inhere$ ls
-file00  -file01  -file02  -file03  -file04  -file05  -file06  -file07  -file08  -file09
```

Ten files, and they all start with a dash, so we already know from [level 1 → 2](/en/posts/bandit-level-1-2/) that we'll need to handle them carefully.

> **Watch out for *human-readable*.** It doesn't mean "with read permissions", it means "containing **text a person can read**". The other nine files hold binary data: open them with `cat` and you'll see garbage characters. Reading the goal correctly is half the level.
{: .prompt-info }

## The question: how do I find it without opening them one by one?

Opening ten files by hand is doable, but it doesn't scale: with a thousand, you wouldn't do it that way. Besides, running `cat` on a binary file can fill your terminal with junk. The level page itself warns you: if your terminal gets messed up, try the `reset` command.

What we need is a tool that tells us **what kind of content** each file has without opening it.

## The key tool: `file`

`file` inspects a file's contents and tells you what it is. It doesn't trust the extension, which means nothing on Linux: it reads the first bytes and matches them against known signatures. That's how it tells apart plain text (`ASCII text`), an image, an executable or raw binary data with no recognisable format (`data`).

## My solution: `find | xargs file`

```console
bandit4@bandit:~/inhere$ find . -type f -readable | xargs file
```

The output shows one line per file with its type. Nine come out as **`data`** and only one as **`ASCII text`**: that's our file.

Breaking it down:

1. **`find .`** searches from the current directory (`.`).
2. **`-type f`** keeps only files, not directories.
3. **`-readable`** keeps only files **our user can read**. It's a **permissions** filter, not a content filter. Here it drops nothing, because we can read all of them; remove it and you'll get the same result. It's useful elsewhere, when you search a whole system and don't want your screen flooded with files you can't open.
4. **`| xargs file`**: `find` prints a list of paths, one per line. `xargs` takes that list and turns it into **arguments** for `file`. In other words, it runs `file ./-file00 ./-file01 ...` with every path at once.

Why do we need `xargs`? Because `file` doesn't read its list of files from standard input: it expects them as arguments. A bare `find ... | file` wouldn't work. `xargs` is the bridge between "one command's output" and "another command's arguments".

And one detail that makes it work: `find` returns paths like `./-file00`, **with `./` in front**. It's the same trick as level 1 → 2, so `file` doesn't mistake the names for options even though they start with a dash.

With the file located, we read it. I won't tell you which one it is, check it yourself:

```console
bandit4@bandit:~/inhere$ cat ./-file0X
[REDACTED]
```

## Other ways to solve it

```console
bandit4@bandit:~/inhere$ file ./*
bandit4@bandit:~/inhere$ find . -type f -exec file {} +
bandit4@bandit:~/inhere$ find . -type f | xargs file | grep text
```

- **`file ./*`**: the shortest. The shell expands `*` to every file in the directory. Notice the `./`: with a plain `file *`, the shell would pass `-file00`, `-file01`… and `file` would read them as options. The level 1 → 2 trick again.
- **`find . -type f -exec file {} +`**: does the same as `xargs` without leaving `find`. `{}` is replaced by the paths found, and `+` tells it to pass them all together in a single run.
- **`... | grep text`**: adds a final filter so only the interesting line shows up. With ten files it doesn't matter, but with a thousand it's the difference between reading one line or a thousand.

> **`xargs` and spaces.** `xargs` splits arguments on whitespace, so it would break on names like the `--spaces in this filename--` from [level 2 → 3](/en/posts/bandit-level-2-3/). The robust version is `find . -type f -print0 | xargs -0 file`, which separates paths with a null character instead of spaces. Or just use `-exec`, which doesn't have that problem.
{: .prompt-tip }

## Moving on to level 5

```console
bandit4@bandit:~/inhere$ exit
$ sshpass -p '[REDACTED]' ssh bandit5@bandit.labs.overthewire.org -p 2220
```

## Summary

| Command / concept | What it does |
|---|---|
| *human-readable* | Content a person can read (text), not "with read permissions" |
| `file somefile` | Identify the content type from its bytes, not its extension |
| `find . -type f` | List every file under a directory |
| `-readable` | Filter by the current user's read permission |
| `xargs command` | Turn one command's output into another's arguments |
| `find ... -exec cmd {} +` | Run a command on the results without `xargs` |
| `reset` | Restore the terminal if it gets garbled after printing binary data |

{% include series-nav.html %}
