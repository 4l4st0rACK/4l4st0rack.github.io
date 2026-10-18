---
title: "OverTheWire Bandit: Level 8 → 9 Walkthrough"
date: 2026-10-18 21:15:00 +0200
categories: [Linux, Bandit]
tags: [bandit, overthewire, linux, sort, uniq, pipes]
description: OverTheWire Bandit level 8 to 9 walkthrough. Find the only line that appears once using sort and uniq -u, and learn why uniq doesn't work without sorting first.
ref: bandit-8-9
serie: bandit
nivel: 9
---

In the [previous level](/en/posts/bandit-level-7-8/) we searched for a line by its **content**: the one with the word `millionth`. Now there's no word to search for. The only clue is **how many times** the line appears: every other line is repeated, and the password is the only one that shows up just once. `grep` won't help us here. Time to meet another classic pair: `sort` and `uniq`.

> Passwords are shown as `[REDACTED]`, following the [OverTheWire rules](https://overthewire.org/rules/).
{: .prompt-warning }

## Connecting to level 8

```console
$ sshpass -p '[REDACTED]' ssh bandit8@bandit.labs.overthewire.org -p 2220
bandit8@bandit:~$ export TERM=xterm
```

## The challenge

According to the [level 9 page](https://overthewire.org/wargames/bandit/bandit9.html), the password is stored in the file `data.txt` and is **the only line that occurs only once**.

```console
bandit8@bandit:~$ ls
data.txt
```

## The solution: sort + uniq -u

```console
bandit8@bandit:~$ cat data.txt | sort | uniq -u
[REDACTED]
```

Three commands chained with pipes, just like in the previous level. Let's see what each one contributes:

| Part | What it does |
|---|---|
| `cat data.txt` | Dumps the contents of the file. |
| `sort` | Sorts the lines alphabetically, so repeated lines end up **next to each other**. |
| `uniq -u` | Shows only the lines that have **no** copy right next to them (*unique*). |

### Why do we need `sort`?

This is the key detail of the level: **`uniq` only compares each line with the one right before it**. It doesn't walk the whole file remembering what it has seen; it only looks at its neighbour. So if two copies of the same line are separated by other lines, `uniq` treats them as different.

A simple example, with a made-up five-line file:

```text
pear
apple
pear
kiwi
apple
```

Here, `uniq -u` without sorting would print all five lines, because none of them has a copy **right next to it**. But if we sort first:

```text
apple
apple
kiwi
pear
pear
```

Now the repeated lines are together, and `uniq -u` leaves only `kiwi`, the one that appears once. Exactly what we need with `data.txt`.

> Rule of thumb: almost every time you use `uniq`, put a `sort` in front of it. Without sorting, the result depends on the order of the file and is rarely what you expect.
{: .prompt-tip }

### `uniq` options

| Command | What it shows |
|---|---|
| `uniq` | One copy of each line (removes consecutive duplicates) |
| `uniq -u` | Only the lines that appear **once** |
| `uniq -d` | Only the lines that are **repeated** (one copy of each) |
| `uniq -c` | Each line prefixed with the **number of times** it appears |

## Other ways to do it

### 1. Without `cat`

```console
bandit8@bandit:~$ sort data.txt | uniq -u
[REDACTED]
```

Just like with `grep` in the previous level, `sort` accepts the file as an argument, so `cat` isn't needed. Same result, one process fewer.

### 2. Counting occurrences with `uniq -c`

```console
bandit8@bandit:~$ sort data.txt | uniq -c | sort -n | head -n 1
```

This route is longer, but it shows you **what's inside** the file:

- `uniq -c` prefixes each line with how many times it appears.
- `sort -n` sorts by that number **numerically** (without `-n`, `sort` would compare as text and `10` would come before `9`).
- `head -n 1` keeps the first line: the one with the lowest count, i.e. the one that appears only once, with a `1` in front of it.

If you drop the `head`, you'll also see how many times each of the other lines is repeated, which is a nice way to understand how the challenge was built.

### 3. With awk alone, no sorting

```console
bandit8@bandit:~$ awk '{c[$0]++} END {for (l in c) if (c[l] == 1) print l}' data.txt
[REDACTED]
```

Back to `awk`, which we met in the previous level, but this time using an **associative array** (a dictionary):

- `{c[$0]++}` runs on every line: it uses the whole line (`$0`) as the key and adds one to its counter.
- `END {...}` runs only once, after the whole file has been read.
- `for (l in c) if (c[l] == 1) print l` loops over every key and prints the ones whose counter is 1.

Unlike `uniq`, `awk` **does remember** every line it has seen, so it doesn't need the file to be sorted.

## Moving on to the next level

```console
bandit8@bandit:~$ exit
$ sshpass -p '[REDACTED]' ssh bandit9@bandit.labs.overthewire.org -p 2220
```

## Summary

| Command / concept | What it's for |
|---|---|
| `sort` | Sort lines, putting repeated ones together |
| `sort -n` | Sort numerically |
| `uniq -u` | Show only the lines that appear once |
| `uniq -c` | Count how many times each line appears |
| `uniq -d` | Show only the repeated lines |
| `head -n 1` | Keep only the first line |
| `awk` with arrays | Count occurrences without sorting |
| `uniq` without `sort` | Only compares consecutive lines: it almost always needs `sort` first |

{% include series-nav.html %}
