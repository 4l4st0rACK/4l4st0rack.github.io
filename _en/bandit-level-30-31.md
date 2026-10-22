---
title: "OverTheWire Bandit: Level 30 → 31 Walkthrough"
date: 2026-10-22 17:58:00 +0200
categories: [Linux, Bandit]
tags: [bandit, overthewire, linux, git, git-tag, git-show]
description: OverTheWire Bandit level 30 to 31 walkthrough, finding the password hidden in a Git tag with git tag and git show.
ref: bandit-30-31
serie: bandit
nivel: 31
---

The last odd corner of Git in Bandit: **tags**. The files are empty, the history seems to have nothing... but the secret is tucked into a tag. It's a place almost nobody checks, and that's exactly why it's worth knowing.

> Level passwords are shown as **[REDACTED]**. Following OverTheWire's rules, I don't publish credentials: the fun part is getting there yourself.
{: .prompt-warning }

## Clone the repo (from your machine)

Like the previous levels, the clone is done **from your local machine**. The credential for `bandit30-git` is bandit30's password.

```console
$ cd $(mktemp -d)
$ git clone ssh://bandit30-git@bandit.labs.overthewire.org:2220/home/bandit30-git/repo
$ cd repo
$ ls
README.md
$ cat README.md
just an empty file... muahaha
```

The README mocks us: it's "empty". The history (`git log`) also has nothing useful, and there are no interesting branches. If the usual places are empty, it's time to look in the less usual ones.

## The concept: Git tags

A **tag** is a named label that points to a specific point in the repository (usually a commit). They're used to mark versions (`v1.0`, `v2.3`...). The important part for us: a tag **can carry its own message and content**, and it doesn't show up in `git log` or `git branch`. A perfect hiding spot.

## The solution

`git tag` lists all the repo's tags:

```console
$ git tag
secret
```

There's a tag called `secret`. Looks like exactly what we're after. `git show <tag>` shows what it points to and its content:

```console
$ git show secret
[REDACTED]
```

And there's bandit31's password. `git show` works with tags just like with commits: it shows you the object the tag points to.

> `git show` is a "Swiss army knife" command: you can give it a commit, a branch, a tag or `commit:file`, and it shows you whatever is there. That's why it's appeared in almost all these Git levels.
{: .prompt-tip }

## Why this matters in security

This level closes an idea we've been carrying since level 27: **a Git repository stores information in far more places than it seems**. Summing up the whole block, when auditing a repo you have to look in:

- The **working tree** (the current files) — level 27 → 28.
- The commit **history** — level 28 → 29.
- All the **branches** — level 29 → 30.
- The **tags** — this level.
- And also the **stash**, the reflogs, loose objects (`git fsck --lost-found`)...

A secret can be in any of those corners. That's why scanning tools (`gitleaks`, `trufflehog`) don't limit themselves to the files: they walk the entire repo. If you hide information in a tag thinking nobody looks there, you're wrong: a `git tag` gives it away.

## Moving to the next level

```console
$ sshpass -p '[REDACTED]' ssh bandit31@bandit.labs.overthewire.org -p 2220
```

> The next level flips Git around: instead of **reading** a repo, you'll have to **push** to it, meeting specific conditions, to make it release the password.
{: .prompt-info }

## Summary

| Command / concept | What it's for |
| --- | --- |
| `git tag` | List all the repository's tags |
| Git tag | Named label pointing to a point in the repo; can hide data |
| `git show <tag>` | Show what a tag points to and its content |
| A repo's corners | Working tree, history, branches, tags, stash, reflog... |

{% include series-nav.html %}
