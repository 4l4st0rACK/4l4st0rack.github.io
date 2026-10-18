---
title: "OverTheWire Bandit: Level 5 → 6 Walkthrough"
date: 2026-10-18 17:00:00 +0200
categories: [Linux, Bandit]
tags: [bandit, overthewire, linux, find, tr]
description: Twenty directories and one hidden file matching three conditions. Combining find filters by size and permissions, and cleaning the output with tr.
ref: bandit-5-6
serie: bandit
nivel: 6
---

On the previous level we searched among ten files. Now the file is hidden in one of **twenty directories**, and the goal gives us its "identikit": three properties it must have. Time to get the most out of `find` by combining several filters at once.

> Passwords show up as `[REDACTED]`, following the [OverTheWire rules](https://overthewire.org/rules/).
{: .prompt-warning }

## Connecting to level 5

```console
$ sshpass -p '[REDACTED]' ssh bandit5@bandit.labs.overthewire.org -p 2220
bandit5@bandit:~$ export TERM=xterm
```

## The goal

The [level 6 page](https://overthewire.org/wargames/bandit/bandit6.html) says the password is in a file somewhere under `inhere` that is:

- ***human-readable***,
- **1033 bytes** in size,
- **not executable**.

```console
bandit5@bandit:~$ ls
inhere
bandit5@bandit:~$ cd inhere
bandit5@bandit:~/inhere$ ls
maybehere00  maybehere01  maybehere02  ...  maybehere19
```

Twenty directories, each with several files inside. Checking them by hand would take forever, but we have the identikit, and `find` knows how to search by properties.

## The fix: `find` with several filters

```console
bandit5@bandit:~/inhere$ find . -type f -size 1033c ! -executable
./maybehere07/.file2
```

A single result. Breaking it down:

- **`find .`**: searches from the current directory, **walking every subdirectory**. No need to go into the twenty directories one by one.
- **`-type f`**: files only.
- **`-size 1033c`**: **exactly 1033 bytes**. The **`c`** matters, more on that below.
- **`! -executable`**: the `!` **negates** the condition after it, so it keeps files that are **not** executable. You can also write `-not -executable`, which reads better.

When you chain several filters, `find` combines them with a **logical AND**: the file has to meet **all** of them at once. That's why, with three conditions, only one file out of dozens is left.

> **The `-size` trap:** with no suffix, `find` doesn't measure in bytes but in **512-byte blocks**. `-size 1033` would look for files around 528 KB. For bytes you need **`c`**. Other handy suffixes: `k` (KiB), `M` (MiB), `G` (GiB). And with `+` or `-` in front you search for "more than" or "less than": `-size +1M` finds files over one megabyte.
{: .prompt-danger }

Also look at the result: **`.file2` starts with a dot**, so it's a hidden file, like on [level 3 → 4](/en/posts/bandit-level-3-4/). `ls` would have hidden it, but **`find` shows hidden files by default**. One more reason to use `find` when you're really looking for something.

### Relative path, not absolute

`./maybehere07/.file2` is a **relative path**: it starts with `./`, meaning "from the directory I'm in". `find` builds paths from the starting point you give it. Since we said `.`, they all start with `./`. If we had run `find /home/bandit5/inhere ...`, we'd have got **absolute** paths, which start with `/` and work from anywhere.

## Reading the file and cleaning the output

```console
bandit5@bandit:~/inhere$ cat ./maybehere07/.file2 | tr -d ' '
[REDACTED]
```

Opening the file with a plain `cat`, the password shows up followed by a **huge blank gap**. That makes sense: the password is tiny and the file has to be 1033 bytes, so it's **padded with spaces** up to that size.

**`tr -d ' '`** cleans it up. `tr` (*translate*) transforms characters, and with **`-d`** (*delete*) it **removes** every character you give it, here the space. The password has no spaces, so it's untouched, and all the padding disappears.

## Other ways to solve it

**Adding the *human-readable* check.** My `find` worked because only one file matched size and permissions, but it doesn't check the third condition. We can do it with `file`, like on [level 4 → 5](/en/posts/bandit-level-4-5/):

```console
bandit5@bandit:~/inhere$ find . -type f -size 1033c ! -executable -exec file {} + | grep text
```

**Find and read in one step.** No need to copy the path: `find` can hand the result straight to `cat`:

```console
bandit5@bandit:~/inhere$ find . -type f -size 1033c ! -executable -exec cat {} + | tr -d ' '
```

**Other ways to strip the padding:**

```console
bandit5@bandit:~/inhere$ cat ./maybehere07/.file2 | sed 's/ //g'
bandit5@bandit:~/inhere$ cat ./maybehere07/.file2 | awk '{print $1}'
```

- **`sed 's/ //g'`**: substitutes (`s`) every space with nothing, across the whole line (`g`, *global*). Same result as `tr -d ' '`, but `sed` handles far more complex substitutions.
- **`awk '{print $1}'`**: prints only the first field of each line. `awk` splits on whitespace, so it keeps the password and drops the rest. It's the sibling of the `$NF` we used on [level 0 → 1](/en/posts/bandit-level-0-1/).

> `-executable` and `-readable` check the permissions **of the user running `find`**, not the file owner's. If you want to filter on exact permission bits, say "executable by anyone", there's the `-perm` option. We'll need it later on.
{: .prompt-tip }

## Moving on to level 6

```console
bandit5@bandit:~/inhere$ exit
$ sshpass -p '[REDACTED]' ssh bandit6@bandit.labs.overthewire.org -p 2220
```

## Summary

| Command / concept | What it does |
|---|---|
| `find . -type f` | Search for files across every subdirectory |
| `-size 1033c` | Exact size in bytes (`c`); no suffix means 512-byte blocks |
| `-size +1M` / `-size -10k` | More than / less than a size |
| `! -executable` / `-not -executable` | Negate a condition |
| Several filters in a row | Combined with AND: all must match |
| `find ... -exec cat {} +` | Read whatever is found directly |
| `tr -d ' '` | Delete every space from the output |
| `sed 's/ //g'` | Substitute text; here, strip the spaces |

{% include series-nav.html %}
