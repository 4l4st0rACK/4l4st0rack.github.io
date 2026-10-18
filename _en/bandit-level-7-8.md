---
title: "OverTheWire Bandit: Level 7 → 8 Walkthrough"
date: 2026-10-18 21:00:00 +0200
categories: [Linux, Bandit]
tags: [bandit, overthewire, linux, grep, awk, pipes]
description: OverTheWire Bandit level 7 to 8 walkthrough. Chain grep and awk with pipes to pull the password next to the word millionth out of a huge text file.
ref: bandit-7-8
serie: bandit
nivel: 8
---

In the [previous level](/en/posts/bandit-level-6-7/) we had to find a file lost somewhere on the whole server. Now we know exactly which file holds the password, but the problem is a different one: that file is **huge**, and what we want is a single line among thousands. Time to meet two essential tools for working with text in Linux: `grep` to filter lines and `awk` to keep just the part of the line we care about.

> Passwords are shown as `[REDACTED]`, following the [OverTheWire rules](https://overthewire.org/rules/).
{: .prompt-warning }

## Connecting to level 7

```console
$ sshpass -p '[REDACTED]' ssh bandit7@bandit.labs.overthewire.org -p 2220
bandit7@bandit:~$ export TERM=xterm
```

## The challenge

According to the [level 8 page](https://overthewire.org/wargames/bandit/bandit8.html), the password is stored in the file `data.txt`, **next to the word `millionth`**.

```console
bandit7@bandit:~$ ls
data.txt
```

If we try opening it straight away with `cat data.txt`, the terminal fills up with line after line: each one has a word followed by a string that looks like a password. Reading it by eye isn't an option. We need to **filter**.

> To get an idea of a file's size before opening it, `wc -l data.txt` tells you how many lines it has. That way you don't flood your terminal.
{: .prompt-tip }

## Filtering with grep

```console
bandit7@bandit:~$ cat data.txt | grep "millionth"
millionth	[REDACTED]
```

Two pieces are at work here:

- **The pipe `|`** connects the standard output (stdout) of the command on the left to the standard input (stdin) of the one on the right. `cat` dumps the whole file, but instead of going to the screen, that text flows into `grep`.
- **`grep "millionth"`** reads line by line and only lets through the lines that contain the pattern `millionth`. Everything else is dropped.

The result: out of the whole file, only the line we want survives, with the word and the password.

## Keeping only the password with awk

The previous line already gives us the password, but the word `millionth` is still in front of it. To get **only** the password, we add one more link to the chain:

```console
bandit7@bandit:~$ cat data.txt | grep "millionth" | awk '{print $2}'
[REDACTED]
```

`awk` is a tool (actually a small programming language) for processing text by columns:

- By default, it splits each line into **fields** using whitespace (spaces and tabs) as the separator.
- `$1` is the first field (the word `millionth`), `$2` is the second (the password), and `$0` would be the whole line.
- `{print $2}` means "for every line you receive, print the second field".

> Notice that the `awk` program goes inside **single quotes**. This matters: with double quotes, the shell would try to replace `$2` with one of its own variables (which would be empty) before `awk` ever saw it, and the command wouldn't do what you expect.
{: .prompt-info }

## Other ways to do it

### 1. Without `cat`: grep can read the file directly

```console
bandit7@bandit:~$ grep "millionth" data.txt
millionth	[REDACTED]
```

`grep` accepts the file name as an argument, so `cat data.txt |` isn't needed. It works the same, but saves you a process. It even has a name: *useless use of cat*. Using `cat` isn't wrong, just redundant.

### 2. With awk alone

```console
bandit7@bandit:~$ awk '/millionth/ {print $2}' data.txt
[REDACTED]
```

`awk` can filter too: the pattern between slashes, `/millionth/`, does `grep`'s job, and the `{print $2}` block only runs on matching lines. A single command does the work of all three.

### 3. With `cut` and whole-word matching

```console
bandit7@bandit:~$ grep -w "millionth" data.txt | cut -f2
[REDACTED]
```

- `grep -w` matches the **whole word**: it wouldn't match something like `millionths` if it existed in the file. It doesn't change the result here, but it's a good habit when you're looking for a specific word.
- `cut -f2` extracts the second field, but **only if the separator is a tab**, which is `cut`'s default (unlike `awk`, which accepts any whitespace). You can check which separator is used with `grep millionth data.txt | cat -A`: tabs show up as `^I`.

## Moving on to the next level

```console
bandit7@bandit:~$ exit
$ sshpass -p '[REDACTED]' ssh bandit8@bandit.labs.overthewire.org -p 2220
```

## Summary

| Command / concept | What it's for |
|---|---|
| Pipe (vertical bar) | Pass one command's output as the next command's input |
| `grep pattern file` | Show only the lines containing the pattern |
| `grep -w` | Match the whole word |
| `awk '{print $2}'` | Print the second field of each line |
| `awk '/pattern/ {print $2}'` | Filter and print a field in one step |
| `cut -f2` | Extract the second tab-separated field |
| `wc -l` | Count the lines in a file |
| Single quotes | Stop the shell from interpreting `$2` before `awk` does |

{% include series-nav.html %}
