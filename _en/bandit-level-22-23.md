---
title: "OverTheWire Bandit: Level 22 → 23 Walkthrough"
date: 2026-10-19 21:14:00 +0200
categories: [Linux, Bandit]
tags: [bandit, overthewire, linux, cron, md5sum, bash, scripting]
description: OverTheWire Bandit level 22 to 23 walkthrough, reading a cron script that computes the file name with md5sum, and reproducing that calculation by hand.
ref: bandit-22-23
serie: bandit
nivel: 23
---

Still on cron, but this time the file where the password lands isn't in plain sight: **its name is computed by the script itself**. To solve it you have to read the script, understand how it generates that name and **reproduce the same calculation** by hand. It's an important step up: we go from "read what's there" to "reason about what a program is going to do".

> Level passwords are shown as **[REDACTED]**. Following OverTheWire's rules, I don't publish credentials: the fun part is getting there yourself.
{: .prompt-warning }

## Connecting

```console
$ sshpass -p '[REDACTED]' ssh bandit22@bandit.labs.overthewire.org -p 2220
bandit22@bandit:~$ export TERM=xterm
```

## The challenge

The [level goal](https://overthewire.org/wargames/bandit/bandit23.html) warns us it's similar to the previous one and sends us back to `/etc/cron.d/`. The config repeats the pattern:

```console
bandit22@bandit:~$ cat /etc/cron.d/cronjob_bandit23
@reboot bandit23 /usr/bin/cronjob_bandit23.sh  &> /dev/null
* * * * * bandit23 /usr/bin/cronjob_bandit23.sh  &> /dev/null
```

Just like level 21 → 22: a task that runs **as bandit23** every minute, executes a script and sends all output to `/dev/null`. The difference is in the script.

## Read the script

```console
bandit22@bandit:~$ cat /usr/bin/cronjob_bandit23.sh
#!/bin/bash

myname=$(whoami)
mytarget=$(echo I am user $myname | md5sum | cut -d ' ' -f 1)

echo "Copying passwordfile /etc/bandit_pass/$myname to /tmp/$mytarget"

cat /etc/bandit_pass/$myname > /tmp/$mytarget
```

Let's go line by line, because that's where all the fun is:

- **`myname=$(whoami)`**: stores the name of the user running the script in the `myname` variable. Since cron launches it as bandit23, here `myname` is `bandit23` (not bandit22!).
- **`mytarget=$(echo I am user $myname | md5sum | cut -d ' ' -f 1)`**: this is the key. It computes the target file name like this:
  - `echo I am user $myname` produces the text `I am user bandit23`.
  - `| md5sum` computes the MD5 hash of that text. `md5sum` returns the hash followed by a dash (`hash  -`).
  - `| cut -d ' ' -f 1` cuts that output using the space as a separator (`-d ' '`) and keeps the first field (`-f 1`), i.e. **just the hash**, without the trailing `  -`.
- **`cat /etc/bandit_pass/$myname > /tmp/$mytarget`**: dumps bandit23's password into `/tmp/<hash>`.

In short: bandit23's password is in a `/tmp` file whose name is the **MD5 of the text `I am user bandit23`**. The catch: the name is computed with `whoami`, which is **bandit23** when cron runs it, not the user we are right now.

> An MD5 is a hash function: it turns any text into a fixed-length string, and for the same input it **always** gives the same output. That's why we can reproduce the calculation: if we use exactly the same text, we get exactly the same file name the script generated.
{: .prompt-info }

## The solution: reproduce the calculation

I compute the hash myself, using the target user's name (bandit23):

```console
bandit22@bandit:~$ echo I am user bandit23 | md5sum | cut -d ' ' -f 1
8ca319486bfbbc3663ea0fbe81326349
```

And now I read the `/tmp` file with that name:

```console
bandit22@bandit:~$ cat /tmp/8ca319486bfbbc3663ea0fbe81326349
[REDACTED]
```

There's bandit23's password.

> Mind the quotes and spaces: the text has to be **exactly** `I am user bandit23`. One extra space, or typing `bandit22` out of habit, gives a different hash and therefore a file that doesn't exist. The hash is deterministic, so you either nail it or it won't work.
{: .prompt-tip }

## Another way: run the script yourself... but no

The temptation is to run the script directly so it does the work. It doesn't work the way you'd expect: if **you** (bandit22) run it, `whoami` returns `bandit22`, it will try to read `/etc/bandit_pass/bandit22` (which you can read, it's yours) and write it to the `/tmp` file for bandit22. **It doesn't give you bandit23's password.** Bandit23's password can only be written by someone who is bandit23, and that's exactly what the cron does for us. That's why the correct route is to let the cron do its job and **recompute the name** ourselves to read its result.

## Moving to the next level

```console
bandit22@bandit:~$ exit
$ sshpass -p '[REDACTED]' ssh bandit23@bandit.labs.overthewire.org -p 2220
```

> The next level is the one we anticipated: a cron that **executes** a script in a directory where you can write. There you no longer read, you **drop your own script** for bandit24 to run.
{: .prompt-info }

## Summary

| Command / concept | What it's for |
| --- | --- |
| `$(command)` | Command substitution: store the output in a variable |
| `whoami` | Show the current user (here, the one running the cron) |
| `md5sum` | Compute the MD5 hash of an input |
| `cut -d ' ' -f 1` | Keep the first field, splitting by spaces |
| Deterministic hash | The same input always gives the same hash: it can be reproduced |
| `cat /etc/bandit_pass/$myname > /tmp/$mytarget` | Dump the password into a file with a computed name |

{% include series-nav.html %}
