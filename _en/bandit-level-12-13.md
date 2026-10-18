---
title: "OverTheWire Bandit: Level 12 → 13 Walkthrough"
date: 2026-10-18 22:10:00 +0200
categories: [Linux, Bandit]
tags: [bandit, overthewire, linux, xxd, hexdump, compression, bash-scripting]
description: OverTheWire Bandit level 12 to 13 walkthrough. Reverse a hexdump with xxd -r, identify each compression layer and write a bash script that keeps decompressing until the password appears.
ref: bandit-12-13
serie: bandit
nivel: 13
---

In the [previous level](/en/posts/bandit-level-11-12/) we only had to undo a single transformation. This level is the first one in Bandit that really takes a while: the file is a **hexdump** (a hexadecimal representation) of another file that has been **compressed many times**, mixing different formats. We need to reverse the hexdump, peel the layers like an onion and, since we don't know how many there are, automate it with a small **bash script**.

> Passwords are shown as `[REDACTED]`, following the [OverTheWire rules](https://overthewire.org/rules/). We don't publish the full hexdump either: with it, anyone could rebuild the file and extract the password.
{: .prompt-warning }

## Connecting to level 12

```console
$ sshpass -p '[REDACTED]' ssh bandit12@bandit.labs.overthewire.org -p 2220
bandit12@bandit:~$ export TERM=xterm
```

## The challenge

According to the [level 13 page](https://overthewire.org/wargames/bandit/bandit13.html), the password is stored in the file `data.txt`, which is a **hexdump of a file that has been repeatedly compressed**.

```console
bandit12@bandit:~$ ls
data.txt
bandit12@bandit:~$ cat data.txt
00000000: 1f8b 0808 b83e b86a 0203 6461 7461 322e  .....>.j..data2.
00000010: 6269 6e00 0146 02b9 fd42 5a68 3931 4159  bin..F...BZh91AY
[...]
```

## How to read a hexdump

Each line has three columns:

| Column | Example | What it is |
|---|---|---|
| Offset | `00000000:` | Position of the line's first byte, in hexadecimal |
| Hex bytes | `1f8b 0808 b83e ...` | 16 bytes per line, grouped in pairs |
| ASCII view | `.....>.j..data2.` | The same bytes as text; non-printable ones show as `.` |

These two lines alone already give us clues:

- **`1f8b`** at the very start is gzip's **signature** (*magic bytes*). Almost every file format starts with a few fixed bytes that identify it, and that's how `file` knows what a file is without looking at its extension.
- The ASCII column shows **`data2.bin`**: gzip stores the original file name in its header.
- And on the second line, **`BZh`** peeks out: the signature of **bzip2**. In other words, we're already seeing the next layer: inside the gzip there's a bzip2.

## From hexdump to file: xxd -r

We copy the whole hexdump to our machine, save it in a file (`hexdump_data`) and turn it back into binary with `xxd`:

```console
$ xxd -r hexdump_data compressed_file
```

`xxd` is the tool that produces hexdumps in exactly this format, and with `-r` (*reverse*) it does the opposite: it reads the hexdump and writes the original bytes. The first argument is the input file and the second is the output file.

Let's check what we got:

```console
$ file compressed_file
compressed_file: gzip compressed data, was "data2.bin", last modified: Sat Sep 26 21:52:56 2026, max compression, from Unix, original size modulo 2^32 582
```

Just what the *magic bytes* suggested: a gzip containing `data2.bin`.

## Looking inside with 7z

```console
$ 7z l compressed_file
[...]
   Date      Time    Attr         Size   Compressed  Name
------------------- ----- ------------ ------------  ------------------------
2026-09-26 23:52:56 .....          582          615  data2.bin
------------------- ----- ------------ ------------  ------------------------
2026-09-26 23:52:56                582          615  1 files
```

`7z l` (*list*) shows the contents of a compressed archive without extracting it. The big advantage of `7z` in this level is that it **understands almost every format** (gzip, bzip2, tar, zip...), so we don't have to worry about which type each layer is: `7z x` (*extract*) can unpack all of them.

If we extract `compressed_file` with `7z x`, we get `data2.bin`, and running `7z l` on it shows... **another** compressed file. We could keep going by hand, layer by layer, but we don't know how many there are. The perfect moment for a script.

## The script: decompress until nothing is left

The script starts from a file called `data.gz`, so first we rename the one we rebuilt:

```console
$ mv compressed_file data.gz
```

And here's the script:

```bash
#!/bin/bash

function ctrl_c(){
  echo -e "\n\n[!] Saliendo...\n"
  exit 1
}

# Ctrl+C
trap ctrl_c INT

first_file_name="data.gz"
decompressed_file_name="$(7z l data.gz | tail -n 3 | head -n 1 | awk 'NF{print $NF}')"

7z x $first_file_name &>/dev/null

while [ $decompressed_file_name ]; do
  echo -e "\n[+] Nuevo archivo descomprimido: $decompressed_file_name"
  7z x $decompressed_file_name &>/dev/null
  decompressed_file_name="$(7z l $decompressed_file_name 2>/dev/null | tail -n 3 | head -n 1 | awk 'NF{print $NF}')"
done
```

(The messages are in Spanish: "Saliendo" means "Exiting" and "Nuevo archivo descomprimido" means "New decompressed file".)

Let's go through it piece by piece.

### Exiting cleanly with Ctrl+C

```bash
function ctrl_c(){ ... }
trap ctrl_c INT
```

When you press Ctrl+C, the system sends the process the **SIGINT** (*interrupt*) signal. `trap` tells bash: "when that signal arrives, instead of just dying, run the `ctrl_c` function". Here the function only prints a message and exits with `exit 1` (an error code), but it's a good habit in any script with loops: if something gets stuck, you exit in a controlled way.

### Getting the name of the file inside

```bash
7z l data.gz | tail -n 3 | head -n 1 | awk 'NF{print $NF}'
```

This line is the heart of the script. Look at the `7z l` output above: the line we care about, the one with the file name, is **the third from the bottom**.

| Step | What it does |
|---|---|
| `tail -n 3` | Keeps the last 3 lines of the listing |
| `head -n 1` | Of those, keeps the first: the file's line |
| `awk 'NF{print $NF}'` | Prints the line's last field: the name |

We already know `$NF` from [level 10 → 11](/en/posts/bandit-level-10-11/): the last field. What's new is the `NF` **before** the braces. In `awk`, whatever comes before `{...}` is a condition: the block only runs if it's true. `NF` is 0 on an empty line, and 0 counts as false, so `NF{print $NF}` means "print the last field, but only if the line isn't empty". That detail is exactly what makes the loop end, as we'll see.

### The loop

```bash
while [ $decompressed_file_name ]; do
  ...
done
```

`[ $variable ]` is true as long as the variable has content. On each iteration the script:

1. Prints the name of the file that just appeared.
2. Extracts it with `7z x`, sending all output to `/dev/null` with `&>` (stdout **and** stderr at once; in [level 6 → 7](/en/posts/bandit-level-6-7/) we only redirected stderr with `2>`).
3. Lists its contents with `7z l` to get the name of the next layer.

When we reach the last file, which is no longer compressed, `7z l` fails: it prints no listing, just an error notice at the end. The lines reaching `head -n 1` are empty, `awk` prints nothing thanks to the `NF` condition, the variable ends up empty and the loop stops.

> A small robustness detail: it's good practice to quote variables, `[ -n "$decompressed_file_name" ]` and `7z x "$decompressed_file_name"`. It works here without quotes because the names have no spaces, but a name like `my file.gz` would break the script.
{: .prompt-tip }

### Running it

```console
$ bash descomprimir.sh

[+] Nuevo archivo descomprimido: data2.bin

[+] Nuevo archivo descomprimido: data2

[+] Nuevo archivo descomprimido: data4.bin

[+] Nuevo archivo descomprimido: data5.bin

[+] Nuevo archivo descomprimido: data6.bin

[+] Nuevo archivo descomprimido: data6

[+] Nuevo archivo descomprimido: data8.bin

[+] Nuevo archivo descomprimido: data9.bin
```

Notice `data2` and `data6`, with no extension: those are bzip2 layers, and unlike gzip, bzip2 **doesn't store the original file name**, so `7z` derives it by stripping the extension.

The last one, `data9.bin`, is already plain text:

```console
$ cat data9.bin
The password is [REDACTED]
```

## Other ways to do it

### 1. Fetch the file with scp instead of copy and paste

```console
$ sshpass -p '[REDACTED]' scp -P 2220 bandit12@bandit.labs.overthewire.org:data.txt hexdump_data
```

Copying and pasting a 40-line hexdump works, but a single miscopied character is enough for `xxd -r` to produce a corrupt file. `scp` copies files over SSH, using the same connection details we already use to log in. Watch out for a detail that trips everyone up: in `ssh` the port goes with lowercase `-p`, but in `scp` it's **uppercase `-P`**.

### 2. Do everything on the server, layer by layer, with `file`

Bandit has `xxd`, `gzip`, `bzip2` and `tar`, so you can solve it without leaving the server. Since we can't write to our home directory, we work in a temporary directory:

```console
bandit12@bandit:~$ cd $(mktemp -d)
bandit12@bandit:/tmp/tmp.XXXXXXXXXX$ cp ~/data.txt .
bandit12@bandit:/tmp/tmp.XXXXXXXXXX$ xxd -r data.txt > data
bandit12@bandit:/tmp/tmp.XXXXXXXXXX$ file data
```

`mktemp -d` creates a randomly named directory in `/tmp` and prints its path, and `$(...)` (command substitution) passes that path straight to `cd` as an argument.

From there, for each layer you check the type with `file` and use the right tool: `gzip -d` for gzip, `bzip2 -d` for bzip2 and `tar -xf` for tar. One catch: `gzip -d` and `bzip2 -d` complain if the file doesn't have the expected extension, so you have to rename it first (`mv data data.gz`). In this file the order of layers is:

| Layer | Format |
|---|---|
| 1 | gzip |
| 2 | bzip2 |
| 3 | gzip |
| 4 | tar |
| 5 | tar |
| 6 | bzip2 |
| 7 | tar |
| 8 | gzip |
| 9 | ASCII text (the password) |

Doing it by hand is the most educational way to understand what happens at each step, even if it's tedious.

### 3. A script that decides with `file`

If you don't have `7z`, you can automate the previous method by letting `file` decide which tool to use for each layer:

```bash
#!/bin/bash
f="data"
while true; do
  type=$(file -b "$f")
  echo "[+] $f: ${type%%,*}"
  case "$type" in
    gzip*)  mv "$f" "$f.gz";  gzip -d "$f.gz" ;;
    bzip2*) mv "$f" "$f.bz2"; bzip2 -d "$f.bz2" ;;
    *tar*)  next=$(tar -tf "$f"); tar -xf "$f"; rm "$f"; f="$next" ;;
    *)      cat "$f"; break ;;
  esac
done
```

- `file -b` (*brief*) prints only the description, without the file name in front.
- `${type%%,*}` cuts off everything from the first comma onwards, to show just the format.
- `case` compares the type against each pattern and runs the matching branch. If it's not gzip, bzip2 or tar, we assume we've reached the end: we show the contents with `cat` and leave the loop with `break`.
- `tar -tf` lists the name of the file inside the tar before extracting it, so we know what the next layer will be called.

It runs on the `data` file produced by `xxd -r` and walks through the same nine layers as the table above.

## Moving on to the next level

```console
bandit12@bandit:~$ exit
$ sshpass -p '[REDACTED]' ssh bandit13@bandit.labs.overthewire.org -p 2220
```

## Summary

| Command / concept | What it's for |
|---|---|
| Hexdump | A hexadecimal view of a file: offset, bytes and ASCII |
| *Magic bytes* | Leading bytes that identify the format (`1f8b` gzip, `BZh` bzip2) |
| `xxd -r input output` | Turn a hexdump back into binary |
| `file` | Identify a file's type from its contents |
| `7z l` / `7z x` | List / extract almost any compressed format |
| `trap function INT` | Run a function when Ctrl+C is pressed |
| `awk 'NF{print $NF}'` | Print the last field only on non-empty lines |
| `&>/dev/null` | Discard stdout and stderr at once |
| `scp -P port` | Copy files over SSH (uppercase `-P`) |
| `cd $(mktemp -d)` | Create and enter a temporary directory |
| `gzip -d` / `bzip2 -d` / `tar -xf` | Decompress each format separately |

{% include series-nav.html %}
