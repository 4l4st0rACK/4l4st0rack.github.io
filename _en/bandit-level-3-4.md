---
title: "OverTheWire Bandit: Level 3 → 4 Walkthrough"
date: 2026-10-18 15:00:00 +0200
categories: [Linux, Bandit]
tags: [bandit, overthewire, linux, ls, cd, hidden-files]
description: A directory that looks empty but isn't. Learn to move around with cd and reveal hidden files with ls -la.
ref: bandit-3-4
serie: bandit
nivel: 4
---

On this level the password sits in a directory that **looks empty** at first glance. It's a chance to learn how to move between directories with `cd` and to discover Linux **hidden files**, which are everywhere and one of the first things worth checking when exploring a system.

> Passwords show up as `[REDACTED]`, following the [OverTheWire rules](https://overthewire.org/rules/).
{: .prompt-warning }

## Connecting to level 3

```console
$ sshpass -p '[REDACTED]' ssh bandit3@bandit.labs.overthewire.org -p 2220
bandit3@bandit:~$ export TERM=xterm
```

## The goal

The [level 4 page](https://overthewire.org/wargames/bandit/bandit4.html) says the password is in a **hidden file** inside the `inhere` directory. Let's see what's in our home:

```console
bandit3@bandit:~$ ls
inhere
```

## Entering the directory: `cd`

To move into `inhere` we use `cd` (*change directory*):

```console
bandit3@bandit:~$ cd inhere
bandit3@bandit:~/inhere$
```

Notice the **prompt changed**: it used to say `~` and now it says `~/inhere`. The prompt always shows which directory you're in, so it's the quickest way to know where you are without running `pwd`.

## First attempt: `ls`

```console
bandit3@bandit:~/inhere$ ls
bandit3@bandit:~/inhere$
```

Nothing. `ls` shows no files. Is it empty? The goal already gave us the hint: the file is **hidden**.

## What a hidden file is in Linux

Linux has no special "hidden" attribute like Windows does. The rule is much simpler: **any file or directory whose name starts with a dot (`.`) is hidden**, and `ls` doesn't show it by default.

It's a convention meant to keep listings free of configuration files. Your home directory is full of them: `.bashrc`, `.profile`, `.ssh/`… You don't see them with a plain `ls`, but they're there.

> Being hidden **doesn't protect a file at all**. Anyone with permission to list the directory can see it with a single `ls` option. That's why checking hidden files is one of the first things you do when exploring a system: it's where configs, histories and sometimes forgotten credentials live.
{: .prompt-danger }

## The fix: `ls -la`

`ls` has options to show hidden files too. The most common combination is `-la`:

```console
bandit3@bandit:~/inhere$ ls -la
```

Among the output lines, our file finally shows up: **`...Hiding-From-You`**. The two options do different things:

- **`-a`** (*all*) shows **every** file, including the ones starting with a dot.
- **`-l`** (*long*) uses the long format, one line per file: permissions, owner, group, size, date and name.

If you only want the names, `ls -a` is cleaner:

```console
bandit3@bandit:~/inhere$ ls -a
.  ..  ...Hiding-From-You
```

Three things show up here, and it's worth not mixing them up:

| Name | What it is |
|---|---|
| `.` | The current directory. We used it on [level 1 → 2](/en/posts/bandit-level-1-2/) with `./-` |
| `..` | The parent directory: `cd ..` takes you up one level |
| `...Hiding-From-You` | Our file. It starts with a dot, so it's hidden |

The name is chosen to trick you: it starts with **three dots**, so at a quick glance it blends in with `.` and `..`.

## Reading the file

```console
bandit3@bandit:~/inhere$ cat ...Hiding-From-You
[REDACTED]
```

No tricks needed this time. The name doesn't start with a dash and has no spaces, so `cat` opens it just fine. The only odd thing about this file is that it's hidden.

## Other ways to solve it

```console
bandit3@bandit:~$ ls -A inhere
bandit3@bandit:~$ cat inhere/...Hiding-From-You
bandit3@bandit:~$ find inhere -type f
```

- **`ls -A`**: like `-a`, but without `.` and `..`. It leaves only the real files, hidden ones included.
- **`cat inhere/...Hiding-From-You`**: you don't need to `cd` in. You can read the file from your home directory using its relative path.
- **`find inhere -type f`**: `find` walks the directory and lists every file (`-type f`), hidden or not. With a single file it makes no difference, but when a directory has subdirectories and hundreds of files, `find` is the tool for the job. We're going to need it a lot in the next levels.

## Moving on to level 4

```console
bandit3@bandit:~/inhere$ exit
$ sshpass -p '[REDACTED]' ssh bandit4@bandit.labs.overthewire.org -p 2220
```

## Summary

| Command / concept | What it does |
|---|---|
| `cd directory` | Move into another directory |
| `cd ..` | Go up to the parent directory |
| Hidden file | Any name starting with `.`; a convention, not protection |
| `ls -a` / `ls -A` | List hidden files too (`-A` without `.` and `..`) |
| `ls -l` | Long format: permissions, owner, size, date |
| `find dir -type f` | Find every file inside a directory |

{% include series-nav.html %}
