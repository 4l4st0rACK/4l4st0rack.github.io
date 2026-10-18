---
title: "OverTheWire Bandit: Level 2 → 3 Walkthrough"
date: 2026-10-18 14:00:00 +0200
categories: [Linux, Bandit]
tags: [bandit, overthewire, linux, cat, bash, quoting]
description: A file with spaces and dashes in its name. How the shell splits arguments on spaces, and how to stop it with quotes, escapes and tab completion.
ref: bandit-2-3
serie: bandit
nivel: 3
---

After the file called `-`, this level raises the stakes: a file named **`--spaces in this filename--`**. It has dashes at the start and **spaces** in the middle, and the spaces are what's going to give us trouble.

> Passwords show up as `[REDACTED]`, following the [OverTheWire rules](https://overthewire.org/rules/).
{: .prompt-warning }

## Connecting to level 2

```console
$ sshpass -p '[REDACTED]' ssh bandit2@bandit.labs.overthewire.org -p 2220
bandit2@bandit:~$ export TERM=xterm
```

## The goal

The [level 3 page](https://overthewire.org/wargames/bandit/bandit3.html) says the password is in a file called `--spaces in this filename--` in the home directory:

```console
bandit2@bandit:~$ ls
--spaces in this filename--
```

## First attempt: the trick from last level

Since the name starts with dashes, it's fair to assume a plain `cat` will mistake it for an option. On the [previous level](/en/posts/bandit-level-1-2/) we fixed that by adding `./` in front, so let's try the same:

```console
bandit2@bandit:~$ cat ./--spaces in this filename--
cat: ./--spaces: No such file or directory
cat: in: No such file or directory
cat: this: No such file or directory
cat: filename--: No such file or directory
```

Four errors, one per word in the name. `cat` tried to open **four different files**: `./--spaces`, `in`, `this` and `filename--`. None of them exist.

## Why it happens: the shell splits on spaces

Before running a command, the **shell** (bash, here) reads the line and splits it into words using spaces as separators. The first word is the command and the rest are its arguments. `cat` doesn't get the line as you typed it, it gets it already split:

| What we typed | What `cat` receives |
|---|---|
| `cat ./--spaces in this filename--` | 4 arguments: `./--spaces`, `in`, `this`, `filename--` |

`cat` has no way of knowing those four words were really one name. It does its job: it tries to open four files. The `./` fixed the dash problem, but spaces are a different problem.

## The fix: quotes

```console
bandit2@bandit:~$ cat ./"--spaces in this filename--"
[REDACTED]
```

**Quotes** tell the shell: "everything in here is **a single word**, even if it has spaces". Now `cat` gets one argument, `./--spaces in this filename--`, and opens the right file.

The quotes don't have to wrap the whole word. They can cover just the tricky part, like here, or everything: `cat "./--spaces in this filename--"` works the same. The shell removes the quotes and glues whatever is on either side into one word.

> **Double or single quotes:** for this case it doesn't matter. The difference is that inside **double** quotes (`"..."`) the shell still interprets some special characters, like `$` for variables, while inside **single** quotes (`'...'`) everything is literal. If a file name contained a `$`, you'd want single quotes.
{: .prompt-info }

## So why do we still need `./`?

Quotes fix the spaces, but not the dashes. Without `./`:

```console
bandit2@bandit:~$ cat "--spaces in this filename--"
cat: unrecognized option '--spaces in this filename--'
```

Now `cat` gets a single word, but it starts with `--`, so it reads it as a long option that doesn't exist. We need both: **quotes for the spaces and `./` for the dashes**.

## Other ways to solve it

```console
bandit2@bandit:~$ cat ./--spaces\ in\ this\ filename--
bandit2@bandit:~$ cat -- "--spaces in this filename--"
```

- **Escaping the spaces with `\`**: the backslash tells the shell the next character is literal, not a separator. It's exactly what bash does when you tab-complete.
- **`--` to end the options**: on the previous level we saw `--` didn't help with a lone dash. Here **it does work**, because the name *starts* with dashes but isn't just a dash. Everything after `--` is treated as a file, even if it starts with `-`.

> **The most practical trick is the Tab key.** Type `cat ./--sp` and press **Tab**. Bash completes the name and escapes the spaces for you. It saves you from typos on weird names, and shows you how an expert would have written it.
{: .prompt-tip }

## Moving on to level 3

```console
bandit2@bandit:~$ exit
$ sshpass -p '[REDACTED]' ssh bandit3@bandit.labs.overthewire.org -p 2220
```

## Summary

| Concept | What we learned |
|---|---|
| Word splitting | The shell splits the line on spaces before running the command |
| `"..."` | Groups into one argument; things like `$` are still interpreted inside |
| `'...'` | Groups into one argument; everything is literal |
| `\ ` | Escapes a space so it doesn't act as a separator |
| `--` | Marks the end of options: what follows is a file even if it starts with `-` |
| Tab | Completes and escapes names for you |

{% include series-nav.html %}
