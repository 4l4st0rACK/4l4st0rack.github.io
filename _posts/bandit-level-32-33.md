---
title: "OverTheWire Bandit: Level 32 → 33 Walkthrough"
date: 2026-10-22 18:45:00 +0200
categories: [Linux, Bandit]
tags: [bandit, overthewire, linux, shell, bash, restricted-shell]
description: How to solve the last OverTheWire Bandit level, 32 to 33, escaping the uppercase shell with the $0 trick.
ref: bandit-32-33
serie: bandit
nivel: 33
---

The last Bandit level, and one of the cleverest. They drop you into an **"uppercase shell"**: a shell that converts everything you type to uppercase, so no command works (`ls` becomes `LS`, which doesn't exist). The challenge is to escape it, and the solution is one of those elegant tricks you never forget once you see it.

> Level passwords are shown as **[REDACTED]**. Following OverTheWire's rules, I don't publish credentials: the fun part is getting there yourself.
{: .prompt-warning }

## The challenge

The [level goal](https://overthewire.org/wargames/bandit/bandit33.html) warns you: after logging in you're dropped into an **uppercase shell**, and your mission is to escape it. On entry, this message and a different prompt greet you:

```console
$ sshpass -p '[REDACTED]' ssh bandit32@bandit.labs.overthewire.org -p 2220
WELCOME TO THE UPPERCASE SHELL
>>
```

If you try to type any command, it uppercases it before running and fails:

```console
>> ls
sh: 1: LS: not found
>> whoami
sh: 1: WHOAMI: not found
```

Everything you type is converted to uppercase, and in Linux commands are lowercase, so you run nothing. You have to find something that **survives** the uppercase conversion and gives you a normal shell.

## The solution: `$0`

```console
>> $0
$ 
```

Just typing `$0` gets us a normal shell. Why does it work?

- **`$0` has no letters.** The shell uppercases the **characters** you type, but `$` and `0` aren't letters: there's nothing to "uppercase". They pass through intact. (Compare with `ls`, which does have letters and ends up as `LS`.)
- **`$0` is a special variable.** In bash/sh, `$0` holds the **name of the shell or script currently running**. In this case, the shell running behind the scenes is `sh`.
- Typing `$0`, the shell **expands** it to `sh` and runs it. That launches **a new instance of a normal `sh`**, without the uppercase filter. We've escaped.

Now we have an ordinary `$` prompt. From here everything works in lowercase:

```console
$ export TERM=xterm
$ whoami
bandit33
```

> The general idea ("what input survives the filter and gives me a shell?") is the same one we saw when escaping the restricted shell in level 25 → 26. The filter changes (there it was `more`, here it's the uppercase conversion), but the mindset is identical: find the crack the filter doesn't cover.
{: .prompt-tip }

## Grab bandit33's password

Now as bandit33, we read its password in case you want to reconnect cleanly:

```console
$ cat /etc/bandit_pass/bandit33
[REDACTED]
```

And with that we enter bandit33's home, where the ending awaits:

```console
$ cd /home/bandit33
$ cat README.txt
Congratulations on solving the last level of this game!
...
```

## Why this matters in security

This level is a perfect example of a **restricted shell escape**, one of the most common scenarios in pentesting. You build a "cage" to limit what a user can do (here, by forcing uppercase; in other cases, an `rbash`, a menu, a pager...), but there's almost always a crack left:

- Inputs the filter doesn't transform (like `$0`, which has no letters).
- Features of the interpreter itself that weren't thought to be blocked (variables, expansions, `exec`...).
- "Harmless" programs that can in turn launch a shell ([GTFOBins](https://gtfobins.github.io/)).

The defensive lesson: a restricted shell **isn't a real sandbox**. If you genuinely want to isolate a user, you need system-level mechanisms (containers, a properly done `chroot`, SELinux/AppArmor, seccomp...), not a text filter you can dodge with two characters.

## End of the game

```console
$ whoami
bandit33
```

And that's Bandit done! You've gone through all 34 levels: from a simple `cat` to shell escapes, privilege escalation, brute force, Git repository archaeology and this last uppercase-shell trick. If you've followed the whole series, you now handle the terminal with a fluency that's the foundation of everything that comes next in offensive security.

> Next stop? The creators themselves recommend moving on to their other wargames. **Natas** (web security) or **Leviathan**/**Narnia** (more binaries and logic) are natural follow-ups. And if the pentesting bug has bitten you, this is the moment to make the jump to platforms like Hack The Box.
{: .prompt-info }

## Summary

| Command / concept | What it's for |
| --- | --- |
| Uppercase shell | Restricted shell that converts input to uppercase |
| `$0` | Special variable: the name of the running shell/script |
| Why `$0` escapes | It has no letters (survives the filter) and expands to `sh` |
| Restricted shell escape | Find the input or feature the filter doesn't cover |
| `cat /etc/bandit_pass/bandit33` | Read the password once you're bandit33 |

{% include series-nav.html %}
