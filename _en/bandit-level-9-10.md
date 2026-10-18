---
title: "OverTheWire Bandit: Level 9 → 10 Walkthrough"
date: 2026-10-18 21:25:00 +0200
categories: [Linux, Bandit]
tags: [bandit, overthewire, linux, strings, grep, binary-files]
description: OverTheWire Bandit level 9 to 10 walkthrough. Use strings to pull readable text out of a binary file and grep to find the password preceded by several equals signs.
ref: bandit-9-10
serie: bandit
nivel: 10
---

In the [previous level](/en/posts/bandit-level-8-9/) the file `data.txt` was plain text, just with lots of repeated lines. This time `data.txt` is something else: a **binary** file, full of bytes that don't represent any readable character. The password is in there, surrounded by garbage. The challenge is to bring out only the readable part, and there's a command with a very fitting name for that: `strings`.

> Passwords are shown as `[REDACTED]`, following the [OverTheWire rules](https://overthewire.org/rules/).
{: .prompt-warning }

## Connecting to level 9

```console
$ sshpass -p '[REDACTED]' ssh bandit9@bandit.labs.overthewire.org -p 2220
bandit9@bandit:~$ export TERM=xterm
```

## The challenge

According to the [level 10 page](https://overthewire.org/wargames/bandit/bandit10.html), the password is stored in the file `data.txt`, in one of the few **human-readable** strings in the file, **preceded by several `=` characters**.

```console
bandit9@bandit:~$ ls
data.txt
```

## The problem: cat on a binary file

If we try `cat data.txt`, the terminal fills up with strange symbols, control characters and gaps. The file isn't broken: `cat` just dumps the bytes as they are, and the terminal tries to draw as characters things that were never text.

> Dumping a binary with `cat` can even mess up your terminal (weird colours, swapped characters), because some bytes are interpreted as control sequences. If that happens, the `reset` command brings it back to normal.
{: .prompt-tip }

Going straight to `grep "=====" data.txt` doesn't work either: when `grep` detects that a file is binary, instead of printing the matching lines it only tells you that there **are** matches, with a message like `binary file matches` (the exact wording depends on the version). We need to turn the binary into text before filtering.

## The solution: strings + grep

```console
bandit9@bandit:~$ strings data.txt | grep "====="
```

`strings` walks through the file byte by byte and extracts every **sequence of printable characters** at least 4 characters long (that's the default minimum). Everything else is ignored. The result is a list of "chunks of text" pulled out of the binary, one per line, which we can now pipe into `grep` just like in previous levels.

`grep "====="` keeps the strings that contain at least five equals signs in a row. With this we already see a few lines starting with `=`, and the password is on **the last one**.

Why five `=` instead of just one? Because with so much binary garbage, it's easy for some string to contain a stray `=` by chance. Requiring several in a row makes the filter much more precise.

## Fine-tuning: keeping only the password

```console
bandit9@bandit:~$ strings data.txt | grep "=====" | tail -n 1 | awk '{print $2}'
[REDACTED]
```

We add two more links to the chain:

| Part | What it does |
|---|---|
| `strings data.txt` | Extracts the readable text from the binary. |
| `grep "====="` | Keeps only the strings with several `=` in a row. |
| `tail -n 1` | Keeps the **last** line, which is the one holding the password. |
| `awk '{print $2}'` | Splits on whitespace and prints the second field: the password without the `=` in front. |

`tail` is the sibling of `head`, which we used in the previous level: instead of the first lines, it shows the **last** ones. And `awk '{print $2}'` is exactly the same trick from [level 7 → 8](/en/posts/bandit-level-7-8/): the equals signs are the first field and the password is the second.

> `tail -n 1` works because, in this file, the password is the last match. It's a valid solution, but it depends on position. The third alternative below gives you a filter that doesn't depend on order.
{: .prompt-info }

## Other ways to do it

### 1. Check the file type with `file`

```console
bandit9@bandit:~$ file data.txt
```

Before opening a file you don't know, `file` tells you what it is by looking at its contents, not its extension. Being called `.txt` doesn't guarantee it's text, and this level proves it. If `file` doesn't say it's text (*ASCII text* or similar), you know `cat` will show you garbage and it's time for `strings`.

### 2. Force grep to treat the binary as text with `-a`

```console
bandit9@bandit:~$ grep -a "=====" data.txt
```

The `-a` (*text*) option tells `grep` to process the file as if it were text even when it detects binary content. Now it does print the matching lines, but with a catch: in a binary, "lines" can be extremely long and full of odd bytes, so the password shows up surrounded by garbage. It works, but `strings` gives a much cleaner result.

### 3. A filter that doesn't depend on order

```console
bandit9@bandit:~$ strings data.txt | grep "=====" | grep -oE '[[:alnum:]]{32}'
[REDACTED]
```

Instead of taking the last line, we search for the password's **shape**: Bandit passwords are 32 alphanumeric characters.

- `-E` turns on extended regular expressions, which let us write `{32}` without escaping the braces.
- `[[:alnum:]]` is any letter or digit, and `{32}` requires exactly 32 in a row.
- `-o` (*only matching*) makes `grep` print **only the matching part**, not the whole line. That's why the `=` disappear without needing `awk`.

As long as only one of the lines contains 32 alphanumeric characters in a row, it doesn't matter where it is: this finds it.

## Moving on to the next level

```console
bandit9@bandit:~$ exit
$ sshpass -p '[REDACTED]' ssh bandit10@bandit.labs.overthewire.org -p 2220
```

## Summary

| Command / concept | What it's for |
|---|---|
| `strings file` | Extract readable text strings from a binary |
| `file filename` | Find out a file's type from its contents |
| `tail -n 1` | Keep only the last line |
| `grep -a` | Treat a binary file as text |
| `grep -oE 'regex'` | Print only the part matching a regular expression |
| `[[:alnum:]]{32}` | Exactly 32 letters or digits in a row |
| `reset` | Fix your terminal if a binary messes it up |

{% include series-nav.html %}
