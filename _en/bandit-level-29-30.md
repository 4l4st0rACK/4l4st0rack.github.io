---
title: "OverTheWire Bandit: Level 29 → 30 Walkthrough"
date: 2026-10-22 17:50:00 +0200
categories: [Linux, Bandit]
tags: [bandit, overthewire, linux, git, git-branch, git-checkout]
description: OverTheWire Bandit level 29 to 30 walkthrough, finding the password in a different branch of the Git repository with git branch and git checkout.
ref: bandit-29-30
serie: bandit
nivel: 30
---

Still on Git, and now **branches** come in. In the previous level the secret was in the main branch's history; here the `master` branch doesn't have it, but the README itself gives a huge hint about where to look. The lesson: a repo isn't just what you see on the default branch.

> Level passwords are shown as **[REDACTED]**. Following OverTheWire's rules, I don't publish credentials: the fun part is getting there yourself.
{: .prompt-warning }

## Clone the repo (from your machine)

Like the previous levels, the clone is done **from your local machine**. The credential for `bandit29-git` is bandit29's password.

```console
$ cd $(mktemp -d)
$ git clone ssh://bandit29-git@bandit.labs.overthewire.org:2220/home/bandit29-git/repo
$ cd repo
$ cat README.md
```

```text
# Bandit Notes
Some notes for bandit30 of bandit.

## credentials

- username: bandit30
- password: <no passwords in production!>
```

## The hint: "no passwords in production!"

Where the password should be it says `<no passwords in production!>`. That's not a bug: it's the hint. It's telling us that **in production** (the `master` branch, the default one) there's no password. And if there's a "production" branch, there are usually others: development, testing... That's most likely where the secret slipped in.

## Look in the branches

`git branch -a` lists **all** branches, including remote ones (`-a` = *all*):

```console
$ git branch -a
* master
  remotes/origin/HEAD -> origin/master
  remotes/origin/dev
  remotes/origin/master
  remotes/origin/sploits-dev
```

Besides `master`, there are two interesting branches: **`dev`** (development) and `sploits-dev`. The development one is the obvious candidate: it's exactly where people put test data (real passwords included) without the care they take in production.

## The solution: switch to the `dev` branch

`git checkout <branch>` switches to that branch, leaving the working directory with its content:

```console
$ git checkout dev
$ ls
code  README.md
```

Here the README is different from the one on `master`. I read it:

```console
$ cat README.md
```

```text
# Bandit Notes
Some notes for bandit30 of bandit.

## credentials

- username: bandit30
- password: [REDACTED]
```

This time the password is in the clear. As always, I refine it to get **only** the password:

```console
$ cat README.md | tail -n 2 | cut -d ':' -f 2
 [REDACTED]
```

Breakdown: `tail -n 2` keeps the last two lines, and `cut -d ':' -f 2` cuts by the colon (`-d ':'`) and keeps the second field (`-f 2`), i.e. whatever comes **after** `password:`. It leaves a leading space, but the password is correct.

> `cut -d ':' -f 2` is the ideal tool when the data comes after a `key: value` style label. On this blog we've already used `awk '{print $NF}'` for the same thing; either works, it's a matter of which you're more comfortable with.
{: .prompt-tip }

## Why this matters in security

The underlying idea connects with the previous level: **a Git repository is much more than its main branch**. Secrets slip in mostly on development branches, where people work with test data and less discipline. When auditing a repo, never stop at `master`/`main`:

- `git branch -a` to see all branches (local and remote).
- Check each one, especially dev/test branches (`dev`, `staging`, `test`...).
- And don't forget the previous level's point: the **history** of each branch, not just its current state.

> That's why tools like `gitleaks` or `trufflehog` scan **everything**: all branches and the full history of each. A credential can be buried in an old commit of a branch nobody looks at.
{: .prompt-tip }

## Moving to the next level

```console
$ sshpass -p '[REDACTED]' ssh bandit30@bandit.labs.overthewire.org -p 2220
```

> The next level closes the Git oddities section with a lesser-known feature: **tags**, another place where information can be hidden in a repository.
{: .prompt-info }

## Summary

| Command / concept | What it's for |
| --- | --- |
| `git branch -a` | List all branches, local and remote |
| `git checkout <branch>` | Switch to another branch of the repository |
| `dev` / `master` branch | Development vs production: the secret is usually in development |
| `cut -d ':' -f 2` | Keep what comes after `label:` |
| Auditing a repo | Check all branches and history, not just the main one |

{% include series-nav.html %}
