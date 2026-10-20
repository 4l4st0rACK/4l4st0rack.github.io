---
title: "OverTheWire Bandit: Level 24 → 25 Walkthrough"
date: 2026-10-20 20:38:00 +0200
categories: [Linux, Bandit]
tags: [bandit, overthewire, linux, netcat, bruteforce, bash, scripting, awk]
description: OverTheWire Bandit level 24 to 25 walkthrough, brute-forcing a 4-digit PIN with a bash script and netcat, then filtering the answer with grep and awk.
ref: bandit-24-25
serie: bandit
nivel: 25
---

First **brute-force** level. There's a service that asks for bandit24's password (which we already have) plus a secret 4-digit PIN we **don't** have. Since there are only 10000 possible PINs, instead of guessing we try them **all** at once with a script. It's your first attack loop, and the concept (automating what's unfeasible by hand) is the heart of half of pentesting.

> Level passwords are shown as **[REDACTED]**. Following OverTheWire's rules, I don't publish credentials: the fun part is getting there yourself.
{: .prompt-warning }

## Connecting

```console
$ sshpass -p '[REDACTED]' ssh bandit24@bandit.labs.overthewire.org -p 2220
bandit24@bandit:~$ export TERM=xterm
```

## The challenge

The [level goal](https://overthewire.org/wargames/bandit/bandit25.html) says there's a **daemon** listening on port **30002** of localhost. To give you bandit25's password it needs you to send, on a single line separated by a space, bandit24's **password** and a secret **4-digit PIN**. There's no way to deduce the PIN: you have to try.

## First contact: talk to the service by hand

Before automating anything, it's worth seeing what the service replies:

```console
bandit24@bandit:~$ nc localhost 30002
I am the pincode checker for user bandit25. Please enter the password for user bandit24 and the secret pincode on a single line, separated by a space.
[REDACTED] 0000
Wrong! Please enter the correct current password and pincode. Try again.
```

I send bandit24's password and a test PIN (`0000`) and it replies `Wrong!`. Perfect: now I know the **format** it expects (`password PIN`) and what it says when I fail. With that I can build the attack: generate the 10000 possible lines and send them all.

> Notice the service doesn't close the connection after a failure: it keeps asking. That's what lets us send all 10000 attempts in a row over the same connection instead of reconnecting each time.
{: .prompt-info }

## The brute-force script

I create a working directory with `mktemp -d` (good practice, like the previous level) and build the script inside:

```bash
#!/usr/bin/env bash

for pin in {0000..9999}; do
    echo "$(cat /etc/bandit_pass/bandit24) ${pin}" >> probando.txt
done

nc localhost 30002 < probando.txt > resultados.txt
```

What it does, part by part:

- **`for pin in {0000..9999}`**: brace expansion that generates every number from 0000 to 9999. The trick is writing `0000` (with the zeros): bash keeps the **zero padding** and generates `0000`, `0001`, ..., `9999`, all 4 digits, which is exactly the PIN format the service wants.
- **`echo "$(cat /etc/bandit_pass/bandit24) ${pin}" >> probando.txt`**: for each PIN, it writes a line with bandit24's password, a space and the PIN, and **appends** it (`>>`) to the `probando.txt` file. When the loop finishes, that file holds all 10000 combinations.
- **`nc localhost 30002 < probando.txt > resultados.txt`**: connects to the daemon and **dumps the whole file** to it over standard input. The service reads line by line, trying each combination, and all its output is saved to `resultados.txt`.

> I use `$(cat /etc/bandit_pass/bandit24)` instead of typing the password directly. It's cleaner (no password left written in the script) and avoids typos. You can read that file because you're bandit24.
{: .prompt-tip }

I run it and check it created the files. A quick look at the size of the response:

```console
bandit24@bandit:/tmp/tmp.XXXX$ bash script.sh
bandit24@bandit:/tmp/tmp.XXXX$ wc -l resultados.txt
```

## Filter the response

`resultados.txt` is full of `Wrong!` lines, one per failed PIN, with the good one lost among them. To find it, I filter **out** the error lines with `grep -v` (the `-v` inverts: it shows what does **not** match):

```console
bandit24@bandit:
