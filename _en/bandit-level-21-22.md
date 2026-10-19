---
title: "OverTheWire Bandit: Level 21 → 22 Walkthrough"
date: 2026-10-19 21:04:00 +0200
categories: [Linux, Bandit]
tags: [bandit, overthewire, linux, cron, cronjob, scheduler, bash]
description: OverTheWire Bandit level 21 to 22 walkthrough, reading a cron job that dumps the password into /tmp, with an introduction to crontab syntax.
ref: bandit-21-22
serie: bandit
nivel: 22
---

Change of topic: **cron jobs** come in, the Linux task scheduler that runs programs automatically at set intervals. In this level we don't have to exploit anything; it's enough to **read what a scheduled task does** and pick up what it leaves behind. But understanding `cron` matters: many real privilege escalations rely on misconfigured scheduled tasks.

> Level passwords are shown as **[REDACTED]**. Following OverTheWire's rules, I don't publish credentials: the fun part is getting there yourself.
{: .prompt-warning }

## Connecting

```console
$ sshpass -p '[REDACTED]' ssh bandit21@bandit.labs.overthewire.org -p 2220
bandit21@bandit:~$ export TERM=xterm
bandit21@bandit:~$ ls
bandit21@bandit:~$
```

Empty home. The clue isn't here, it's in the system configuration.

## The challenge

The [level goal](https://overthewire.org/wargames/bandit/bandit22.html) says a program is running automatically at regular intervals from **cron**, the time-based job scheduler. It tells us to look in `/etc/cron.d/` for the configuration and see what command is being executed.

## Step 1: read the cron configuration

```console
bandit21@bandit:~$ ls /etc/cron.d/
... cronjob_bandit22 ...
bandit21@bandit:~$ cat /etc/cron.d/cronjob_bandit22
@reboot bandit22 /usr/bin/cronjob_bandit22.sh &> /dev/null
* * * * * bandit22 /usr/bin/cronjob_bandit22.sh &> /dev/null
```

You need to know how to read these two lines:

- **`@reboot`**: runs the command once, every time the machine boots.
- **`* * * * *`**: these are cron's five time fields. Five asterisks mean "every minute, of every hour, of every day...". In other words, **the task runs once per minute**.
- **`bandit22`**: the user whose permissions the task runs with. This is key: the script runs **as bandit22**, not as us.
- **`/usr/bin/cronjob_bandit22.sh`**: the script that runs.
- **`&> /dev/null`**: redirects both standard output (stdout) and errors (stderr) to `/dev/null`, i.e. **throws all output away**. That's why we see nothing on screen even though it runs every minute.

Cron's five fields, left to right, are:

| Field | Meaning | Range |
| --- | --- | --- |
| 1 | Minute | 0-59 |
| 2 | Hour | 0-23 |
| 3 | Day of month | 1-31 |
| 4 | Month | 1-12 |
| 5 | Day of week | 0-7 (0 and 7 = Sunday) |

A `*` in a field means "all values". So `* * * * *` = every minute.

## Step 2: read the script

```console
bandit21@bandit:~$ cat /usr/bin/cronjob_bandit22.sh
#!/bin/bash
chmod 644 /tmp/<file>
cat /etc/bandit_pass/bandit22 > /tmp/<file>
```

What the script does, running **as bandit22**:

1. `chmod 644 /tmp/<file>`: sets world-readable permissions (`rw-r--r--`) on a file in `/tmp`.
2. `cat /etc/bandit_pass/bandit22 > /tmp/<file>`: dumps bandit22's password into that file.

In other words, every minute bandit22 writes its own password into a `/tmp` file that **anyone can read**. We don't need to run anything or wait: the file is already there (the task has been running for a while).

## The solution

Just read that file:

```console
bandit21@bandit:~$ cat /tmp/<file>
[REDACTED]
```

And there's bandit22's password.

> The file name in `/tmp` is a long random string. I leave it as `<file>` on purpose: copy it exactly as you see it in **your** output of the script's `cat`, because it changes every time the game resets.
{: .prompt-info }

## Why this matters in security

Here the cron only leaks a password, but the pattern "a privileged task touches files in a path I have access to" is one of the most exploited privilege escalation routes in the real world. If instead of **reading** a file in `/tmp`, the cron **executed** a script you have write permission over, you could insert your own commands and have them run with that user's privileges (exactly what happens in the next level).

That's why, when auditing a machine, reviewing crons is almost mandatory:

```console
bandit21@bandit:~$ ls -la /etc/cron.d/ /etc/cron.daily/ /etc/cron.hourly/
bandit21@bandit:~$ cat /etc/crontab
```

> Enumeration tools like `linpeas` check all this automatically, but it's worth knowing how to do it by hand to understand what the tool is looking at.
{: .prompt-tip }

## Moving to the next level

```console
bandit21@bandit:~$ exit
$ sshpass -p '[REDACTED]' ssh bandit22@bandit.labs.overthewire.org -p 2220
```

> The next level stays with cron but raises the bar: you'll have to **read a script** and work out what file name it generates before you can grab the password.
{: .prompt-info }

## Summary

| Command / concept | What it's for |
| --- | --- |
| `cron` | Time-based task scheduler |
| `/etc/cron.d/` | Directory with system cron configurations |
| `* * * * *` | The 5 time fields: here, "every minute" |
| `@reboot` | Run at machine boot |
| `&> /dev/null` | Discard stdout and stderr |
| `chmod 644` | `rw-r--r--` permissions (readable by everyone) |
| `cat file > destination` | Dump a file's contents into another |

{% include series-nav.html %}
