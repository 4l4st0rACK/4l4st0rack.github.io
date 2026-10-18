---
title: "OverTheWire Bandit: Level 11 → 12 Walkthrough"
date: 2026-10-18 21:50:00 +0200
categories: [Linux, Bandit]
tags: [bandit, overthewire, linux, rot13, tr, caesar-cipher]
description: OverTheWire Bandit level 11 to 12 walkthrough. Learn what ROT13 is, why the same command encodes and decodes it, and undo it with tr to recover the password.
ref: bandit-11-12
serie: bandit
nivel: 12
---

In the [previous level](/en/posts/bandit-level-10-11/) the password was base64-encoded. This time the disguise is even older: every letter in the file has been **shifted 13 positions** in the alphabet. This is known as **ROT13**, a variant of the Caesar cipher, and we'll undo it with a command we've already met: `tr`.

> Passwords are shown as `[REDACTED]`, following the [OverTheWire rules](https://overthewire.org/rules/). We also hide the ROT13 version of the password, because reversing it is instant.
{: .prompt-warning }

## Connecting to level 11

```console
$ sshpass -p '[REDACTED]' ssh bandit11@bandit.labs.overthewire.org -p 2220
bandit11@bandit:~$ export TERM=xterm
```

## The challenge

According to the [level 12 page](https://overthewire.org/wargames/bandit/bandit12.html), the password is stored in the file `data.txt`, where **all lowercase and uppercase letters have been rotated by 13 positions**.

```console
bandit11@bandit:~$ ls
data.txt
bandit11@bandit:~$ cat data.txt
Gur cnffjbeq vf [REDACTED]
```

`Gur cnffjbeq vf` means nothing... until we apply ROT13 and it turns into `The password is`.

## What is ROT13?

ROT13 (*rotate by 13*) replaces each letter with the one **13 positions further along** in the English alphabet, wrapping back to the start when it reaches the end:

| Original | A | B | C | ... | M | N | O | ... | Z |
|---|---|---|---|---|---|---|---|---|---|
| ROT13 | N | O | P | ... | Z | A | B | ... | M |

It only affects letters: digits, spaces and symbols stay the same. And it preserves case: an `a` becomes `n` and an `A` becomes `N`.

It has a neat property: since the alphabet has 26 letters and 13 is exactly half, **applying ROT13 twice gives you back the original text**. In other words, encoding and decoding are exactly the same operation.

> Just like base64, ROT13 **protects nothing**: there's no key and anyone can undo it in a second. It was (and still is) used to hide spoilers or puzzle answers, not for security.
{: .prompt-danger }

## Undoing ROT13 with tr

```console
bandit11@bandit:~$ cat data.txt | tr 'A-Za-z' 'N-ZA-Mn-za-m'
The password is [REDACTED]
```

`tr` (*translate*) replaces characters one by one: the first set says **which** characters to change and the second says **into what**. The character in position 1 of the first set becomes the one in position 1 of the second, 2 becomes 2, and so on.

Let's break down the sets:

| First set | Becomes | Result |
|---|---|---|
| `A-Z` (A to Z) | `N-ZA-M` (N to Z, then A to M) | A→N, B→O ... M→Z, N→A ... Z→M |
| `a-z` (a to z) | `n-za-m` (n to z, then a to m) | a→n, b→o ... m→z, n→a ... z→m |

The trick is `N-ZA-M`: it's the same 26 letters, but **starting at N**. Pairing them position by position with `A-Z` shifts every letter by 13 places. Same for lowercase.

We already used `tr` in [level 5 → 6](/en/posts/bandit-level-5-6/) to clean up output. Here we see its most classic use: translating one set of characters into another.

## Keeping only the password

```console
bandit11@bandit:~$ cat data.txt | tr 'A-Za-z' 'N-ZA-Mn-za-m' | awk '{print $NF}'
[REDACTED]
```

The same `$NF` from the previous level: the sentence is once again `The password is ...`, and `$NF` prints the last field.

## Other ways to do it

### 1. Without `cat`, but with redirection (not as an argument)

In earlier levels we said `cat file |` was unnecessary because `grep`, `sort` or `base64` accept the file as an argument. **`tr` is the exception**: it only reads from standard input. If you try to pass it the file, it fails:

```console
bandit11@bandit:~$ tr 'A-Za-z' 'N-ZA-Mn-za-m' data.txt
tr: extra operand 'data.txt'
Try 'tr --help' for more information.
```

So here `cat` actually makes sense. If you want to drop it, the right way is **input redirection** with `<`, which connects the file to `tr`'s stdin:

```console
bandit11@bandit:~$ tr 'A-Za-z' 'N-ZA-Mn-za-m' < data.txt
The password is [REDACTED]
```

It's the counterpart of the `2>` redirection we saw in [level 6 → 7](/en/posts/bandit-level-6-7/): there we sent stderr into a file, here we feed a file into stdin.

### 2. Checking that ROT13 is its own inverse

```console
$ echo "hola" | tr
