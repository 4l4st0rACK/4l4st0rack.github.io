---
title: "OverTheWire Bandit: Level 1 → 2 Walkthrough"
date: 2026-10-18 13:00:00 +0200
categories: [Linux, Bandit]
tags: [bandit, overthewire, linux, cat, stdin, paths]
description: A file simply called "-". Why cat won't open it, what standard input is, and how to read it using relative paths.
ref: bandit-1-2
serie: bandit
nivel: 2
---

On this level the password is stored in a file with a very unusual name: **`-`**, a single dash. It looks trivial, but it's a great way to understand how the terminal reads arguments and what **standard input** is.

> Passwords show up as `[REDACTED]`, following the [OverTheWire rules](https://overthewire.org/rules/).
{: .prompt-warning }

## Connecting to level 1

We log in with the password from the [previous level](/en/posts/bandit-level-0-1/) and fix the terminal, as on every level:

```console
$ sshpass -p '[REDACTED]' ssh bandit1@bandit.labs.overthewire.org -p 2220
bandit1@bandit:~$ export TERM=xterm
```

## The goal

The [level 2 page](https://overthewire.org/wargames/bandit/bandit2.html) says the password is in a file called **`-`** in the home directory. Let's check:

```console
bandit1@bandit:~$ ls
-
```

There it is. So we just do the same as on level 0, right?

## Why `cat -` doesn't work

```console
bandit1@bandit:~$ cat -
```

The command prints nothing and just sits there, as if it had frozen. It's not an error: `cat` is doing exactly what we asked.

Linux has a convention followed by a huge number of programs: **a lone dash (`-`) as an argument means "read from standard input"** (*stdin*) instead of a file. When nothing is piped in, standard input is your keyboard. So `cat -` is waiting for you to type something so it can echo it back.

Type anything and press Enter, and `cat` will repeat it. To get out, press **Ctrl+C** (cancel) or **Ctrl+D** (end of input).

That convention exists so commands can be chained. For example, `echo hello | cat -` makes `cat` read whatever comes through the pipe. The problem is that our file is named exactly like that reserved word.

## The fix: `cat ./-`

```console
bandit1@bandit:~$ cat ./-
[REDACTED]
```

The key is that `./-` is no longer a lone dash, it's a **path**:

- `.` means "the directory I'm in right now".
- `/` is the directory separator.
- `-` is the file name.

`./-` reads as "the file called `-` in this directory". It points to exactly the same file as `-`, but since the argument is no longer a bare dash, `cat` doesn't mistake it for standard input and opens it as a regular file.

> The same trick works for any file whose name starts with a dash. A file called `-v` would be taken as an option by many commands. With `./-v` the problem goes away.
{: .prompt-tip }

## Other ways to read it

There's more than one way to solve it, and each one teaches something:

```console
bandit1@bandit:~$ cat /home/bandit1/-
bandit1@bandit:~$ cat < -
```

- **`cat /home/bandit1/-`**: same idea as `./-`, but with the **absolute path**, starting from the filesystem root.
- **`cat < -`**: here `cat` gets no arguments at all. It's the **shell** that opens the file `-` with the `<` redirection and hands it to `cat` as standard input. Since the shell doesn't apply the dash convention, it opens the file just fine.

> You might think of `cat -- -`, since `--` is used to say "options end here". **It doesn't work here.** `--` helps with names that *start* with a dash, like `-v`, but a lone dash still means standard input even after `--`.
{: .prompt-info }

## Moving on to level 2

```console
bandit1@bandit:~$ exit
$ sshpass -p '[REDACTED]' ssh bandit2@bandit.labs.overthewire.org -p 2220
```

## Summary

| Concept | What we learned |
|---|---|
| `-` as an argument | Many programs read it as "read from standard input" |
| Standard input (*stdin*) | What a program receives: the keyboard or whatever comes through a pipe |
| `Ctrl+C` / `Ctrl+D` | Cancel a command / signal end of input |
| `./file` | Path relative to the current directory |
| `/home/user/file` | Absolute path, from the root |
| `command < file` | The shell opens the file and passes it as standard input |

{% include series-nav.html %}
