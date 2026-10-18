---
title: "OverTheWire Bandit: Level 17 → 18 Walkthrough"
date: 2026-10-18 23:28:00 +0200
categories: [Linux, Bandit]
tags: [bandit, overthewire, linux, diff, grep, comm, ssh]
description: OverTheWire Bandit level 17 to 18 walkthrough, finding the only line that changed between two files with diff, grep and comm.
ref: bandit-17-18
serie: bandit
nivel: 18
---

After two networking levels, we're back to files. This level is about a very common task: **comparing two versions of a file and finding what changed**. It's what Git does when it shows you a commit, and the classic Linux tool for it is `diff`.

> Level passwords are shown as **[REDACTED]**. Following OverTheWire's rules, I don't publish credentials: the fun part is getting there yourself.
{: .prompt-warning }

## Connecting

In the [previous level](/en/posts/bandit-level-16-17/) we didn't get a password but an SSH private key, so I log in with it:

```console
$ ssh -i bandit17.key bandit17@bandit.labs.overthewire.org -p 2220
bandit17@bandit:~$ export TERM=xterm
```

## The challenge

The [level goal](https://overthewire.org/wargames/bandit/bandit18.html) says there are two files in the home directory, `passwords.old` and `passwords.new`, and that the password for the next level is in `passwords.new`: **it's the only line that changed between the two**.

```console
bandit17@bandit:~$ ls
passwords.new  passwords.old
```

Each file has a bunch of lines that look like passwords. Comparing them by eye would be madness, so we let the computer do it.

## The solution: `diff`

```console
bandit17@bandit:~$ diff passwords.old passwords.new
<n>c<n>
< [REDACTED]
---
> [REDACTED]
```

There's the password. But you need to know how to read `diff`'s output, because there are **two** redacted lines and only one is the right one:

- `<n>c<n>`: the change header. The number is the affected line (in each file) and the `c` means *change*: that line was **modified**. An added line would show an `a` (*add*) and a deleted one a `d` (*delete*).
- `<`: lines from the **first** file we passed (`passwords.old`).
- `---`: separator between the old and new versions.
- `>`: lines from the **second** file (`passwords.new`).

Since the password is in `passwords.new`, **the right one is the line starting with `>`**.

> Argument order matters. If you run `diff passwords.new passwords.old`, the symbols flip and the password would be the `<` line. A trick to keep it straight: think `diff old new`, just like Git shows a change "from before to after".
{: .prompt-tip }

## Other ways to do it

### 1. `diff` with other output formats

`diff` can show the change in more readable ways:

```console
bandit17@bandit:~$ diff -u passwords.old passwords.new
```

The **unified** format (`-u`) is the one you'll see in Git and in patches: the old line appears with `-` and the new one with `+`, with a few lines of context around them.

```console
bandit17@bandit:~$ diff -y --suppress-common-lines passwords.old passwords.new
```

With `-y` it puts both files **side by side**, and `--suppress-common-lines` hides identical lines. What's left is the old line on the left, the new one on the right and a `|` in between marking the change.

### 2. `grep` using a file as a list of patterns

Another way to think about it: I want the lines in `passwords.new` that **aren't** in `passwords.old`. `grep` can do that:

```console
bandit17@bandit:~$ grep -vxFf passwords.old passwords.new
[REDACTED]
```

Each option is there for a reason:

| Option | What it does |
| --- | --- |
| `-f passwords.old` | Uses each line of `passwords.old` as a search pattern |
| `-F` | Treats patterns as literal text, not regular expressions |
| `-x` | Only counts as a match if the whole line is identical |
| `-v` | Inverts the result: shows lines that **don't** match |

Together: "show me the lines in `passwords.new` that aren't exactly equal to any line in `passwords.old`". And the password comes out directly, no symbols to interpret.

### 3. `comm` with sorted files

`comm` compares two **sorted** files and prints three columns: lines only in the first, only in the second, and common to both. Since our files aren't sorted, I sort them on the fly with `sort`, like in the `sort | uniq` level:

```console
bandit17@bandit:~$ comm -13 <(sort passwords.old) <(sort passwords.new)
[REDACTED]
```

- `<(sort file)` is **process substitution**: bash runs `sort` and hands its output to `comm` as if it were a file.
- `-13` hides column 1 (only in old) and column 3 (common), so only column 2 is left: what's **only in the new file**.

## Moving to the next level

```console
bandit17@bandit:~$ exit
$ sshpass -p '[REDACTED]' ssh bandit18@bandit.labs.overthewire.org -p 2220
```

> Heads up for the next level: its goal already warns that someone modified bandit18's `.bashrc` to **log you out as soon as you connect** over SSH. So don't be surprised if the connection closes instantly: that's the level 18 → 19 challenge.
{: .prompt-info }

## Summary

| Command / concept | What it's for |
| --- | --- |
| `diff old new` | Show the differences between two files |
| `<` / `>` in `diff` | Line from the first file / line from the second |
| `c`, `a`, `d` | Line changed, added or deleted |
| `diff -u` | Unified format (the one Git and patches use) |
| `diff -y --suppress-common-lines` | See only the changes, side by side |
| `grep -vxFf A B` | Lines in B that don't appear whole in A |
| `comm -13 <(sort A) <(sort B)` | Lines that are only in B |
| `<(command)` | Process substitution: use a command's output as a file |

{% include series-nav.html %}
