---
title: "OverTheWire Bandit: Level 28 → 29 Walkthrough"
date: 2026-10-22 17:40:00 +0200
categories: [Linux, Bandit]
tags: [bandit, overthewire, linux, git, git-log, git-show, git-diff]
description: OverTheWire Bandit level 28 to 29 walkthrough, recovering a password that was removed from the README but still lives in the Git commit history.
ref: bandit-28-29
serie: bandit
nivel: 29
---

The previous level was Git for beginners: the secret was in the README. This one takes the important step: the password is **no longer** in the current README, someone "censored" it... but Git **remembers everything**. We're going to recover it from the commit history. It's, literally, one of the most common real-world mistakes: believing that deleting a secret from a file removes it, when it's still in the repo's history.

> Level passwords are shown as **[REDACTED]**. Following OverTheWire's rules, I don't publish credentials: the fun part is getting there yourself.
{: .prompt-warning }

## Clone the repo (from your machine)

Just like the [previous level](/en/posts/bandit-level-27-28/): the clone is done **from your local machine**, not from inside Bandit (otherwise you hit the localhost block). The credential for `bandit28-git` is bandit28's password.

```console
$ cd $(mktemp -d)
$ git clone ssh://bandit28-git@bandit.labs.overthewire.org:2220/home/bandit28-git/repo
$ cd repo
$ cat README.md
```

The README has a credentials section, but the password appears **masked** with `xxxxxxxxxx`. It's not in the current version. Time to look back.

## Dig into the history

`git log` shows the commit history, from most recent to oldest:

```console
$ git log
commit 4357a6e... (HEAD -> master, origin/master, origin/HEAD)
    fix info leak

commit fac3dcd...
    add missing data

commit a0e6303...
    initial commit of README.md
```

The commit messages already tell the story:

- `initial commit of README.md`: the original README.
- `add missing data`: someone **added** data (probably the real password).
- `fix info leak`: someone realized the password was exposed and **masked** it. This is the suspect commit: "fix info leak" almost always means "I removed something that shouldn't have been there".

## The solution: see what that commit changed

`git show <commit>` shows exactly what changed in a commit. I look at the `fix info leak` one:

```console
$ git show 4357a6e76c600eb95daf13b1c315b95faab9d4e6
...
diff --git a/README.md b/README.md
@@ -4,5 +4,5 @@ Some notes for level29 of bandit.
 ## credentials

 - username: bandit29
- password: [REDACTED]
+ password: xxxxxxxxxx
```

And here's the key: you need to know how to **read a diff**.

- Lines starting with **`-`** are what that commit **removed** (the old version).
- Lines starting with **`+`** are what it **added** (the new version).

The `fix info leak` commit **deleted** the line with the real password (`-`) and **replaced** it with `xxxxxxxxxx` (`+`). So the password we want is exactly the one on the `-` line. The attempt to "fix the leak" only changed the file in the new commit; the previous version, with the password, **is still stored in the history**.

> That's the whole lesson of the level: in Git, modifying or deleting something doesn't remove it, it creates a new commit. Everything before is still accessible. "Fixing" a leak by deleting the secret in a later commit **does nothing** if the repo is accessible: anyone with the history recovers it.
{: .prompt-tip }

## Other ways to see it

### 1. The diff between the previous commit and the current one

Instead of `git show`, you can compare the two versions directly:

```console
$ git diff fac3dcd 4357a6e
```

It shows the same change: the password in red (`-`) and the `xxxxxxxxxx` in green (`+`).

### 2. Read the README of an old commit directly

Since we know `add missing data` (the middle commit) already had the password and it hadn't been masked yet, we can read the file **as it was** in that commit:

```console
$ git show fac3dcd:README.md
```

The syntax `git show <commit>:<file>` shows a file's content in the state of a specific commit. There the password is in the clear, no diff to interpret.

## Why this matters in security

This is one of the most common real-world leaks: someone commits a credential, notices, makes another commit "removing it" and rests easy. But the secret stays in the history forever. The **correct** way to react to a leaked secret is not to delete it in a new commit, but to:

1. **Rotate the secret** (change the password/token: anything leaked must be considered burned).
2. And, if needed, rewrite the history (`git filter-repo`, BFG) to purge it for real.

In a pentest, finding a repo (or an exposed `.git`) and reviewing its `git log` is gold. Tools like `gitleaks` or `trufflehog` walk the entire history looking for exactly this kind of secret.

## Moving to the next level

```console
$ sshpass -p '[REDACTED]' ssh bandit29@bandit.labs.overthewire.org -p 2220
```

> The next level stays with Git, but the secret isn't in the linear history: you'll have to look in the repository's **branches** (`git branch`).
{: .prompt-info }

## Summary

| Command / concept | What it's for |
| --- | --- |
| `git log` | View the commit history and its messages |
| `git show <commit>` | See exactly what a commit changed |
| `-` / `+` in a diff | Removed line (old version) / added line (new version) |
| `git diff A B` | Compare two commits |
| `git show <commit>:<file>` | See a file as it was in a commit |
| Immutable history | Deleting a secret in a new commit doesn't remove it from the past |
| Rotate the secret | The correct reaction to a leaked credential |

{% include series-nav.html %}
