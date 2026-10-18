---
title: "OverTheWire Bandit: Level 10 → 11 Walkthrough"
date: 2026-10-18 21:40:00 +0200
categories: [Linux, Bandit]
tags: [bandit, overthewire, linux, base64, encoding, awk]
description: OverTheWire Bandit level 10 to 11 walkthrough. Learn what base64 is (and why it isn't encryption), decode the password with base64 -d and grab the last word with awk and $NF.
ref: bandit-10-11
serie: bandit
nivel: 11
---

In the [previous level](/en/posts/bandit-level-9-10/) the password was hidden among binary garbage. Now it's in plain sight, but **in disguise**: the file holds the password encoded in **base64**, a way of representing data you'll run into constantly in cybersecurity (HTTP headers, tokens, emails, payloads...). Recognising it and decoding it should become second nature.

> Passwords are shown as `[REDACTED]`, following the [OverTheWire rules](https://overthewire.org/rules/). In this level we also hide the base64 string, because anyone could decode it and get the password.
{: .prompt-warning }

## Connecting to level 10

```console
$ sshpass -p '[REDACTED]' ssh bandit10@bandit.labs.overthewire.org -p 2220
bandit10@bandit:~$ export TERM=xterm
```

## The challenge

According to the [level 11 page](https://overthewire.org/wargames/bandit/bandit11.html), the password is stored in the file `data.txt`, which contains **base64-encoded data**.

```console
bandit10@bandit:~$ ls
data.txt
bandit10@bandit:~$ cat data.txt
[REDACTED: base64 string]
```

Opening it shows a single line of letters, digits and a couple of `=` signs at the end. That look alone is a very strong hint that we're dealing with base64.

## What is base64?

Base64 is an **encoding**: a way of representing any data (text, images, binaries) using only 64 "safe" characters that travel without problems through systems designed for text:

- the letters `A-Z` and `a-z`
- the digits `0-9`
- the symbols `+` and `/`
- and `=` as **padding** at the end

It works by taking the data 3 bytes at a time and representing each group with 4 characters. When there isn't a full group of 3 bytes left at the end, it pads with `=`. That's why so many base64 strings end in `=` or `==`, and why they're a bit longer than the original (33% longer).

An example with harmless text:

```console
$ echo "hola" | base64
aG9sYQo=
$ echo "aG9sYQo=" | base64 -d
hola
```

> Base64 **is not encryption**. There's no key and no secret: anyone can decode it with a single command. If you ever see passwords "protected" with base64 in an application or a config file, they're effectively in plain text.
{: .prompt-danger }

## Decoding the password

We copy the string that `cat` showed us and pass it to `base64 -d` (*decode*) with `echo` and a pipe:

```console
bandit10@bandit:~$ echo "[REDACTED: base64 string]" | base64 -d
The password is [REDACTED]
```

The result is an English sentence with the password at the end.

## Keeping only the password: `$NF`

To get just the password, we add `awk` to the chain:

```console
bandit10@bandit:~$ echo "[REDACTED: base64 string]" | base64 -d | awk '{print $NF}'
[REDACTED]
```

In [level 7 → 8](/en/posts/bandit-level-7-8/) we used `$2` to print the second field. Here we use something more flexible:

- `NF` (*Number of Fields*) is an `awk` variable holding **how many fields** the current line has. In `The password is [REDACTED]` there are 4 fields, so `NF` is 4.
- `$NF` is therefore `$4`: **the last field**, whatever its position.

The advantage of `$NF` over typing `$4` by hand is that you don't have to count the words: if the sentence were longer or shorter, it would still return the last one.

## Other ways to do it

### 1. Decode the file directly (no copy and paste)

```console
bandit10@bandit:~$ base64 -d data.txt
The password is [REDACTED]
```

`base64 -d` accepts the file as an argument, just like `grep` or `sort` in previous levels. It's the cleanest option and it also avoids the main risk of the `echo` approach: **making a mistake when copying** the string (one missing character and the decoding comes out wrong or fails). Whenever the data is already in a file, it's better to work on the file.

With the final filter it looks like this:

```console
bandit10@bandit:~$ base64 -d data.txt | awk '{print $NF}'
[REDACTED]
```

### 2. With `cut` and a delimiter

```console
bandit10@bandit:~$ base64 -d data.txt | cut -d ' ' -f 4
[REDACTED]
```

In level 7 → 8 we saw that `cut` uses tab as its default separator. With `-d ' '` we tell it to split on **spaces**, and with `-f 4` to extract the fourth field. It works, but here you do need to know the password is the fourth word: exactly the rigidity that `$NF` avoids.

### 3. A curious detail: the trailing `Cg==`

The string in this level ends in `Cg==`. That's no coincidence: `Cg==` is exactly what base64 produces when encoding a lone **newline**. You can check it yourself:

```console
$ printf '\n' | base64
Cg==
```

That means the original text ended with a newline, like any line created with `echo`. Look at the earlier example too: `echo "hola"` gives `aG9sYQo=`, but if you drop the newline with `printf 'hola'` you get `aG9sYQ==`. Changing a single byte changes the output.

## Moving on to the next level

```console
bandit10@bandit:~$ exit
$ sshpass -p '[REDACTED]' ssh bandit11@bandit.labs.overthewire.org -p 2220
```

## Summary

| Command / concept | What it's for |
|---|---|
| Base64 | An encoding (not encryption) that represents data with 64 characters |
| `base64` | Encode data in base64 |
| `base64 -d file` | Decode base64 straight from a file |
| Trailing `=` / `==` | Base64 padding: a hint for recognising it |
| `awk '{print $NF}'` | Print the last field of the line |
| `NF` | Number of fields in the current line in awk |
| `cut -d ' ' -f 4` | Extract the fourth space-separated field |

{% include series-nav.html %}
